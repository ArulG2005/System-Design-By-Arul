# Single Responsibility Principle (SRP)

## The First SOLID Principle

> **"A class should have only one reason to change."**
> — Robert C. Martin (Uncle Bob)

---

## What Does This Mean?

Every class should have **ONE job** and **ONE job only**.

If a class does multiple things, it has multiple reasons to change. When requirements change, you'll have to modify this class more often, increasing the risk of bugs.

---

## Simple Analogy

Think of a restaurant:
- **Chef** → Only cooks food
- **Waiter** → Only serves customers
- **Cashier** → Only handles payments
- **Cleaner** → Only cleans the place

What if ONE person did all these jobs?
- Hard to manage
- Mistakes everywhere
- Can't scale
- Can't replace or train

The same applies to classes in your code!

---

## Real-World Example: User Management

### ❌ BAD: Violating SRP

```java
// This class does TOO MANY things!
class User {
    private String name;
    private String email;
    
    // Responsibility 1: User data
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    
    // Responsibility 2: Database operations
    public void saveToDatabase() {
        Connection conn = DriverManager.getConnection("jdbc:mysql://localhost/db");
        PreparedStatement stmt = conn.prepareStatement(
            "INSERT INTO users (name, email) VALUES (?, ?)"
        );
        stmt.setString(1, this.name);
        stmt.setString(2, this.email);
        stmt.executeUpdate();
    }
    
    public static User loadFromDatabase(int id) {
        // Database loading logic
    }
    
    // Responsibility 3: Email operations
    public void sendWelcomeEmail() {
        Properties props = new Properties();
        props.put("mail.smtp.host", "smtp.gmail.com");
        Session session = Session.getInstance(props);
        Message message = new MimeMessage(session);
        message.setRecipient(Message.RecipientType.TO, new InternetAddress(email));
        message.setSubject("Welcome!");
        message.setText("Welcome " + name);
        Transport.send(message);
    }
    
    // Responsibility 4: Validation
    public boolean isValidEmail() {
        return email != null && email.contains("@") && email.contains(".");
    }
    
    // Responsibility 5: Formatting
    public String toJson() {
        return "{\"name\": \"" + name + "\", \"email\": \"" + email + "\"}";
    }
    
    public String toXml() {
        return "<user><name>" + name + "</name><email>" + email + "</email></user>";
    }
}
```

### Problems with This Code:

1. **5 reasons to change this class**:
   - User fields change → Modify class
   - Database schema changes → Modify class
   - Email provider changes → Modify class
   - Validation rules change → Modify class
   - JSON/XML format changes → Modify class

2. **Hard to test**: Testing email requires database setup
3. **Hard to reuse**: Can't use email logic elsewhere
4. **Tight coupling**: Everything depends on everything

---

### ✅ GOOD: Following SRP

```java
// Responsibility 1: Only user data
class User {
    private String name;
    private String email;
    
    public User(String name, String email) {
        this.name = name;
        this.email = email;
    }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
}

// Responsibility 2: Only database operations
class UserRepository {
    private Connection connection;
    
    public UserRepository(Connection connection) {
        this.connection = connection;
    }
    
    public void save(User user) {
        PreparedStatement stmt = connection.prepareStatement(
            "INSERT INTO users (name, email) VALUES (?, ?)"
        );
        stmt.setString(1, user.getName());
        stmt.setString(2, user.getEmail());
        stmt.executeUpdate();
    }
    
    public User findById(int id) {
        // Load from database
    }
    
    public List<User> findAll() {
        // Load all users
    }
    
    public void delete(int id) {
        // Delete user
    }
}

// Responsibility 3: Only email operations
class EmailService {
    private String smtpHost;
    private Session session;
    
    public EmailService(String smtpHost) {
        this.smtpHost = smtpHost;
        Properties props = new Properties();
        props.put("mail.smtp.host", smtpHost);
        this.session = Session.getInstance(props);
    }
    
    public void sendWelcomeEmail(User user) {
        Message message = new MimeMessage(session);
        message.setRecipient(Message.RecipientType.TO, 
            new InternetAddress(user.getEmail()));
        message.setSubject("Welcome!");
        message.setText("Welcome " + user.getName());
        Transport.send(message);
    }
    
    public void sendPasswordReset(User user, String resetLink) {
        // Send password reset email
    }
}

// Responsibility 4: Only validation
class UserValidator {
    public boolean isValid(User user) {
        return isValidEmail(user.getEmail()) && isValidName(user.getName());
    }
    
    public boolean isValidEmail(String email) {
        return email != null && email.contains("@") && email.contains(".");
    }
    
    public boolean isValidName(String name) {
        return name != null && name.length() >= 2;
    }
}

// Responsibility 5: Only formatting/serialization
class UserSerializer {
    public String toJson(User user) {
        return "{\"name\": \"" + user.getName() + 
               "\", \"email\": \"" + user.getEmail() + "\"}";
    }
    
    public String toXml(User user) {
        return "<user><name>" + user.getName() + 
               "</name><email>" + user.getEmail() + "</email></user>";
    }
    
    public User fromJson(String json) {
        // Parse JSON to User
    }
}
```

### How to Use These Classes Together:

```java
class UserService {
    private UserRepository repository;
    private UserValidator validator;
    private EmailService emailService;
    
    public UserService(UserRepository repository, 
                       UserValidator validator, 
                       EmailService emailService) {
        this.repository = repository;
        this.validator = validator;
        this.emailService = emailService;
    }
    
    public void registerUser(String name, String email) {
        User user = new User(name, email);
        
        // Validate
        if (!validator.isValid(user)) {
            throw new IllegalArgumentException("Invalid user data");
        }
        
        // Save
        repository.save(user);
        
        // Send welcome email
        emailService.sendWelcomeEmail(user);
    }
}
```

---

## Another Example: Invoice System

### ❌ BAD: One Class Does Everything

```java
class Invoice {
    private List<Item> items;
    private double total;
    
    public void addItem(Item item) { 
        items.add(item); 
    }
    
    public void calculateTotal() {
        total = 0;
        for (Item item : items) {
            total += item.getPrice() * item.getQuantity();
        }
    }
    
    // Printing logic - WRONG!
    public void printInvoice() {
        System.out.println("=== INVOICE ===");
        for (Item item : items) {
            System.out.println(item.getName() + ": $" + item.getPrice());
        }
        System.out.println("Total: $" + total);
    }
    
    // Database logic - WRONG!
    public void saveToDatabase() {
        // SQL operations...
    }
}
```

### ✅ GOOD: Separate Classes

```java
// Only invoice data and calculations
class Invoice {
    private List<Item> items = new ArrayList<>();
    private double total;
    
    public void addItem(Item item) {
        items.add(item);
        recalculateTotal();
    }
    
    public void removeItem(Item item) {
        items.remove(item);
        recalculateTotal();
    }
    
    private void recalculateTotal() {
        total = 0;
        for (Item item : items) {
            total += item.getPrice() * item.getQuantity();
        }
    }
    
    public List<Item> getItems() { return items; }
    public double getTotal() { return total; }
}

// Only printing
class InvoicePrinter {
    public void print(Invoice invoice) {
        System.out.println("=== INVOICE ===");
        for (Item item : invoice.getItems()) {
            System.out.println(item.getName() + ": $" + item.getPrice());
        }
        System.out.println("Total: $" + invoice.getTotal());
    }
    
    public void printDetailed(Invoice invoice) {
        // Detailed format
    }
    
    public void printForCustomer(Invoice invoice) {
        // Customer-friendly format
    }
}

// Only database operations
class InvoiceRepository {
    public void save(Invoice invoice) {
        // Save to database
    }
    
    public Invoice findById(int id) {
        // Load from database
    }
}

// Only PDF generation
class InvoicePdfGenerator {
    public byte[] generate(Invoice invoice) {
        // Generate PDF bytes
    }
}
```

---

## How to Identify SRP Violations

### Red Flags 🚩

1. **Class name includes "And"**
   - `UserAndOrderManager` → Split into `UserManager` and `OrderManager`

2. **Class has many unrelated methods**
   - Methods for database, email, validation, formatting all in one class

3. **Class changes for multiple reasons**
   - "I need to change this class for email AND for database"

4. **Large classes (500+ lines)**
   - Usually doing too many things

5. **Hard to name the class**
   - If you struggle to name it, it probably does too much

6. **"Manager" or "Handler" in the name**
   - `DataManager`, `ProcessHandler` - often do too much

---

## Questions to Ask

When designing a class, ask:

1. **"What is this class responsible for?"**
   - If you use "AND" in your answer, split it.

2. **"Why would this class change?"**
   - List ALL reasons. If more than one, split it.

3. **"Can I describe this class in one sentence without using 'and' or 'or'?"**
   - If no, it does too much.

---

## Common Misconceptions

### ❌ Wrong: "One method per class"
```java
// TOO EXTREME! Not what SRP means
class UserNameGetter {
    public String get(User user) { return user.getName(); }
}
class UserEmailGetter {
    public String get(User user) { return user.getEmail(); }
}
```

### ✅ Right: "One responsibility per class"
```java
// Repository has ONE responsibility: data access
// But it can have many related methods
class UserRepository {
    public void save(User user) { }
    public User findById(int id) { }
    public List<User> findAll() { }
    public void delete(int id) { }
    public User findByEmail(String email) { }
    public void update(User user) { }
}
```

---

## Real-World Patterns Using SRP

### MVC Pattern
```
Model      → Data only
View       → Display only
Controller → Coordination only
```

### Repository Pattern
```
Repository → Database operations only
Service    → Business logic only
Controller → HTTP handling only
```

### Clean Architecture Layers
```
Entity      → Business objects only
Use Case    → Business rules only
Repository  → Data access only
Controller  → Web framework only
```

---

## Benefits of SRP

| Benefit | Explanation |
|---------|-------------|
| **Easier testing** | Test each responsibility in isolation |
| **Easier maintenance** | Changes affect only one class |
| **Better reusability** | Small classes can be reused in other projects |
| **Easier understanding** | Each class has a clear purpose |
| **Parallel development** | Different developers can work on different classes |
| **Fewer bugs** | Changes have limited impact |

---

## Practice Exercise

Refactor this class following SRP:

```java
class Employee {
    private String name;
    private double salary;
    
    // Data
    public String getName() { return name; }
    public double getSalary() { return salary; }
    
    // Tax calculation - Should this be here?
    public double calculateTax() {
        return salary * 0.3;
    }
    
    // Saving to database - Should this be here?
    public void save() {
        // INSERT INTO employees...
    }
    
    // Generate pay slip - Should this be here?
    public void printPaySlip() {
        System.out.println("Name: " + name);
        System.out.println("Salary: " + salary);
        System.out.println("Tax: " + calculateTax());
        System.out.println("Net: " + (salary - calculateTax()));
    }
}
```

**Solution hint**: Create separate classes for:
- `Employee` (data only)
- `TaxCalculator` (tax logic)
- `EmployeeRepository` (database)
- `PaySlipGenerator` (printing)

---

## Summary

- **SRP means**: One class = One responsibility = One reason to change
- **Benefits**: Easier testing, maintenance, reusability
- **How to apply**: Split classes that do multiple things
- **Remember**: Responsibility ≠ Method. A class can have many methods for ONE responsibility.

---

**Next: Open/Closed Principle (OCP) →**
