# Software Design Principles

## What are Software Design Principles?

**Software Design Principles** are guidelines that help developers write code that is:
- Easy to understand
- Easy to maintain
- Easy to extend
- Less prone to bugs

These principles are the foundation of good software design.

---

## Why Do We Need Design Principles?

### Without Principles (Bad Code)
```java
// Everything in one class - nightmare to maintain!
class DoEverything {
    public void processOrder() { /* ... */ }
    public void sendEmail() { /* ... */ }
    public void calculateTax() { /* ... */ }
    public void generateReport() { /* ... */ }
    public void updateDatabase() { /* ... */ }
    // 1000 more methods...
}
```

### With Principles (Good Code)
```java
class OrderProcessor { /* handles orders */ }
class EmailService { /* sends emails */ }
class TaxCalculator { /* calculates tax */ }
class ReportGenerator { /* generates reports */ }
class DatabaseService { /* database operations */ }
```

---

## Core Design Principles

### 1. DRY - Don't Repeat Yourself

**Definition**: Every piece of knowledge should have a single, unambiguous representation.

**Bad Example (Repetition)**:
```java
class UserService {
    public void createUser(String name, String email) {
        // Validation logic
        if (email == null || !email.contains("@")) {
            throw new IllegalArgumentException("Invalid email");
        }
        // Create user...
    }
    
    public void updateUser(String name, String email) {
        // Same validation logic repeated!
        if (email == null || !email.contains("@")) {
            throw new IllegalArgumentException("Invalid email");
        }
        // Update user...
    }
}
```

**Good Example (DRY)**:
```java
class UserService {
    private void validateEmail(String email) {
        if (email == null || !email.contains("@")) {
            throw new IllegalArgumentException("Invalid email");
        }
    }
    
    public void createUser(String name, String email) {
        validateEmail(email);  // Reuse!
        // Create user...
    }
    
    public void updateUser(String name, String email) {
        validateEmail(email);  // Reuse!
        // Update user...
    }
}
```

---

### 2. KISS - Keep It Simple, Stupid

**Definition**: Keep your code as simple as possible. Don't over-engineer.

**Bad Example (Over-Engineered)**:
```java
// Too complex for a simple task!
interface NumberAdder {
    int add(int a, int b);
}

class NumberAdderFactory {
    public static NumberAdder createAdder() {
        return new ConcreteNumberAdder();
    }
}

class ConcreteNumberAdder implements NumberAdder {
    @Override
    public int add(int a, int b) {
        return a + b;
    }
}

// Usage
NumberAdder adder = NumberAdderFactory.createAdder();
int result = adder.add(5, 3);
```

**Good Example (KISS)**:
```java
// Simple and clean!
public int add(int a, int b) {
    return a + b;
}

// Usage
int result = add(5, 3);
```

---

### 3. YAGNI - You Aren't Gonna Need It

**Definition**: Don't add functionality until you actually need it.

**Bad Example (Unnecessary Features)**:
```java
class User {
    private String name;
    private String email;
    private String address;      // Do we need this now?
    private String phoneNumber;  // Do we need this now?
    private Date birthday;       // Do we need this now?
    private String nickname;     // Do we need this now?
    private String twitterHandle; // Do we need this now?
    private int loyaltyPoints;   // Do we need this now?
    
    // Constructor with 10 parameters...
    // 20 getter/setter methods...
}
```

**Good Example (YAGNI)**:
```java
// Start with only what you need
class User {
    private String name;
    private String email;
    
    public User(String name, String email) {
        this.name = name;
        this.email = email;
    }
    
    // Getters and setters for these only
}

// Add more fields WHEN you actually need them
```

---

### 4. Separation of Concerns (SoC)

**Definition**: Different parts of the code should handle different concerns/responsibilities.

**Bad Example (Mixed Concerns)**:
```java
class OrderController {
    public void processOrder(Order order) {
        // Concern 1: Validation
        if (order.getItems().isEmpty()) {
            throw new Exception("No items");
        }
        
        // Concern 2: Database
        Connection conn = DriverManager.getConnection("...");
        PreparedStatement stmt = conn.prepareStatement("INSERT...");
        stmt.executeUpdate();
        
        // Concern 3: Email
        Properties props = new Properties();
        props.put("mail.smtp.host", "smtp.gmail.com");
        Session session = Session.getInstance(props);
        Message message = new MimeMessage(session);
        Transport.send(message);
        
        // Concern 4: Logging
        System.out.println("Order processed: " + order.getId());
    }
}
```

**Good Example (SoC)**:
```java
// Each class handles ONE concern
class OrderValidator {
    public void validate(Order order) {
        if (order.getItems().isEmpty()) {
            throw new ValidationException("No items");
        }
    }
}

class OrderRepository {
    public void save(Order order) {
        // Database logic only
    }
}

class EmailService {
    public void sendOrderConfirmation(Order order) {
        // Email logic only
    }
}

class OrderLogger {
    public void log(Order order) {
        // Logging logic only
    }
}

class OrderController {
    private OrderValidator validator;
    private OrderRepository repository;
    private EmailService emailService;
    private OrderLogger logger;
    
    public void processOrder(Order order) {
        validator.validate(order);
        repository.save(order);
        emailService.sendOrderConfirmation(order);
        logger.log(order);
    }
}
```

---

### 5. Composition over Inheritance

**Definition**: Prefer using object composition instead of inheriting from a base class.

**Problem with Inheritance**:
```java
// Inheritance creates tight coupling
class Bird {
    public void fly() {
        System.out.println("Flying...");
    }
}

class Penguin extends Bird {
    // Problem! Penguins can't fly!
    // But we inherit fly() method
}
```

**Good Example (Composition)**:
```java
// Composition is flexible
interface Flyable {
    void fly();
}

interface Swimmable {
    void swim();
}

class FlyingBehavior implements Flyable {
    public void fly() {
        System.out.println("Flying...");
    }
}

class SwimmingBehavior implements Swimmable {
    public void swim() {
        System.out.println("Swimming...");
    }
}

class Sparrow {
    private Flyable flyBehavior = new FlyingBehavior();
    
    public void fly() {
        flyBehavior.fly();
    }
}

class Penguin {
    private Swimmable swimBehavior = new SwimmingBehavior();
    
    public void swim() {
        swimBehavior.swim();
    }
    // No fly method - Penguins can't fly!
}
```

---

### 6. Program to Interface, Not Implementation

**Definition**: Depend on abstractions (interfaces), not concrete classes.

**Bad Example (Depends on Implementation)**:
```java
class OrderService {
    private MySQLDatabase database;  // Tightly coupled to MySQL!
    
    public OrderService() {
        this.database = new MySQLDatabase();
    }
    
    public void saveOrder(Order order) {
        database.save(order);
    }
}
```

**Good Example (Depends on Interface)**:
```java
interface Database {
    void save(Object obj);
}

class MySQLDatabase implements Database {
    public void save(Object obj) { /* MySQL logic */ }
}

class MongoDatabase implements Database {
    public void save(Object obj) { /* MongoDB logic */ }
}

class OrderService {
    private Database database;  // Depends on interface!
    
    public OrderService(Database database) {
        this.database = database;
    }
    
    public void saveOrder(Order order) {
        database.save(order);  // Works with ANY database!
    }
}

// Usage - easily switch databases
OrderService mysqlService = new OrderService(new MySQLDatabase());
OrderService mongoService = new OrderService(new MongoDatabase());
```

---

### 7. Law of Demeter (Don't Talk to Strangers)

**Definition**: A method should only talk to its immediate friends.

**Bad Example (Violates Law of Demeter)**:
```java
class Customer {
    private Address address;
}

class Address {
    private City city;
}

class City {
    private String zipCode;
}

// Violation - reaching deep into object chain
String zipCode = customer.getAddress().getCity().getZipCode();
```

**Good Example (Follows Law of Demeter)**:
```java
class Customer {
    private Address address;
    
    public String getZipCode() {
        return address.getZipCode();
    }
}

class Address {
    private City city;
    
    public String getZipCode() {
        return city.getZipCode();
    }
}

class City {
    private String zipCode;
    
    public String getZipCode() {
        return zipCode;
    }
}

// Clean - only one level of method call
String zipCode = customer.getZipCode();
```

---

### 8. Encapsulation

**Definition**: Hide internal details and expose only what's necessary.

**Bad Example (No Encapsulation)**:
```java
class BankAccount {
    public double balance;  // Anyone can access and modify!
}

// Dangerous!
BankAccount account = new BankAccount();
account.balance = -1000000;  // Invalid state!
```

**Good Example (Encapsulation)**:
```java
class BankAccount {
    private double balance;  // Hidden!
    
    public double getBalance() {
        return balance;
    }
    
    public void deposit(double amount) {
        if (amount > 0) {
            balance += amount;
        }
    }
    
    public boolean withdraw(double amount) {
        if (amount > 0 && amount <= balance) {
            balance -= amount;
            return true;
        }
        return false;
    }
}

// Safe!
BankAccount account = new BankAccount();
account.deposit(5000);
account.withdraw(1000);
// Can't set invalid balance directly
```

---

### 9. High Cohesion, Low Coupling

**Cohesion**: How closely related the functions within a module are.
**Coupling**: How dependent modules are on each other.

**Goal**: High Cohesion + Low Coupling

**Bad Example (Low Cohesion, High Coupling)**:
```java
// Low cohesion - this class does too many unrelated things
class Utils {
    public String formatDate(Date date) { }
    public double calculateTax(double amount) { }
    public void sendEmail(String to, String body) { }
    public User parseUserFromJson(String json) { }
}

// High coupling - tightly dependent
class OrderProcessor {
    private InventoryManager inventory;
    private PaymentGateway payment;
    private EmailService email;
    private SMSService sms;
    private LoggingService logger;
    private AnalyticsService analytics;
    // Too many dependencies!
}
```

**Good Example (High Cohesion, Low Coupling)**:
```java
// High cohesion - each class has one focused purpose
class DateFormatter {
    public String format(Date date, String pattern) { }
    public Date parse(String dateString, String pattern) { }
}

class TaxCalculator {
    public double calculate(double amount, TaxType type) { }
    public double calculateGST(double amount) { }
}

// Low coupling - minimal dependencies
class OrderProcessor {
    private OrderService orderService;  // Only what's needed
    
    public void process(Order order) {
        orderService.save(order);
    }
}
```

---

### 10. Fail Fast

**Definition**: Detect and report errors as soon as possible.

**Bad Example (Fail Late)**:
```java
class UserRegistration {
    public void register(String email, String password, int age) {
        // Does a lot of work first...
        createAccount();
        sendWelcomeEmail();
        setupProfile();
        
        // Then validates at the end!
        if (email == null) {
            throw new Exception("Email is required");  // Undo everything?
        }
    }
}
```

**Good Example (Fail Fast)**:
```java
class UserRegistration {
    public void register(String email, String password, int age) {
        // Validate FIRST!
        if (email == null || email.isEmpty()) {
            throw new IllegalArgumentException("Email is required");
        }
        if (password == null || password.length() < 8) {
            throw new IllegalArgumentException("Password must be 8+ chars");
        }
        if (age < 18) {
            throw new IllegalArgumentException("Must be 18+");
        }
        
        // Then do the work
        createAccount();
        sendWelcomeEmail();
        setupProfile();
    }
}
```

---

## Summary Table

| Principle | Remember As | Key Point |
|-----------|-------------|-----------|
| DRY | Don't Repeat | One place for each piece of logic |
| KISS | Keep Simple | Don't over-engineer |
| YAGNI | Not Now | Add features when needed |
| SoC | Separate | Each module handles one concern |
| Composition | Has-A | Prefer composition over inheritance |
| Program to Interface | Use Abstractions | Depend on interfaces |
| Law of Demeter | Direct Friends Only | Don't chain method calls |
| Encapsulation | Hide Details | Private fields, public methods |
| Cohesion/Coupling | Together/Apart | Related things together, minimal dependencies |
| Fail Fast | Check Early | Validate at the start |

---

## How These Principles Connect

```
DRY + KISS + YAGNI
        ↓
    Clean Code
        ↓
   SoC + Encapsulation
        ↓
   Modular Design
        ↓
Composition + Interfaces
        ↓
  Flexible Systems
        ↓
     SOLID Principles (Next Chapter!)
```

---

## Practice Exercise

Take this messy code and apply the principles:

```java
// Before: Violates many principles
class Shop {
    public void sell(String item, int qty, String customerEmail) {
        // Check stock
        Connection conn = DriverManager.getConnection("jdbc:mysql://localhost/shop");
        Statement stmt = conn.createStatement();
        ResultSet rs = stmt.executeQuery("SELECT stock FROM items WHERE name='" + item + "'");
        int stock = rs.getInt("stock");
        
        if (stock < qty) {
            System.out.println("Not enough stock");
            return;
        }
        
        // Update stock
        stmt.executeUpdate("UPDATE items SET stock = stock - " + qty + " WHERE name='" + item + "'");
        
        // Send email
        Properties props = new Properties();
        props.put("mail.smtp.host", "smtp.gmail.com");
        Session session = Session.getInstance(props);
        Message msg = new MimeMessage(session);
        msg.setRecipient(Message.RecipientType.TO, new InternetAddress(customerEmail));
        msg.setSubject("Order Confirmation");
        msg.setText("You bought " + qty + " " + item);
        Transport.send(msg);
        
        System.out.println("Sale complete");
    }
}
```

**Challenge**: Refactor this code applying DRY, SoC, and Encapsulation!

---

**Next: SOLID Principles - Single Responsibility Principle (SRP) →**
