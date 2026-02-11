# API Design Best Practices

## Why Good API Design Matters

Your API is a **contract** between your code and its users. A well-designed API:
- Is easy to understand and use correctly
- Hard to misuse
- Self-documenting
- Consistent and predictable

---

## 1. Method Naming

### Use Clear, Descriptive Names

```java
// BAD: Unclear names ❌
void process(Order o);
void doIt();
Data get();
void handle(Request r);

// GOOD: Self-explanatory ✅
void processPayment(Order order);
void validateUserCredentials();
Order findOrderById(String orderId);
void handlePaymentWebhook(PaymentRequest request);
```

### Naming Conventions

| Operation | Prefix | Example |
|-----------|--------|---------|
| Create | `create`, `add`, `register` | `createUser()`, `addItem()` |
| Read | `get`, `find`, `fetch` | `getUser()`, `findByEmail()` |
| Update | `update`, `modify`, `set` | `updateProfile()`, `setStatus()` |
| Delete | `delete`, `remove`, `cancel` | `deleteOrder()`, `removeItem()` |
| Check | `is`, `has`, `can` | `isActive()`, `hasPermission()` |
| Convert | `to`, `as`, `from` | `toJson()`, `asString()` |

---

## 2. Method Parameters

### Avoid Too Many Parameters

```java
// BAD: Too many parameters ❌
User createUser(String name, String email, String phone, 
                String address, String city, String country,
                int age, boolean active);

// GOOD: Use a parameter object ✅
User createUser(UserCreationRequest request);

class UserCreationRequest {
    private String name;
    private String email;
    private String phone;
    private Address address;
    private int age;
    private boolean active;
    // Getters and setters
}
```

### Use Builders for Complex Objects

```java
// Fluent builder pattern
User user = User.builder()
    .name("John")
    .email("john@example.com")
    .phone("123-456-7890")
    .address(new Address("123 Main St", "NYC", "USA"))
    .age(30)
    .active(true)
    .build();
```

### Avoid Boolean Parameters (Use Enums)

```java
// BAD: Boolean is unclear ❌
void sendEmail(String to, String subject, boolean urgent);
sendEmail("john@example.com", "Meeting", true);  // What's true?

// GOOD: Enum is self-documenting ✅
enum Priority { LOW, NORMAL, HIGH, URGENT }

void sendEmail(String to, String subject, Priority priority);
sendEmail("john@example.com", "Meeting", Priority.URGENT);
```

---

## 3. Return Types

### Never Return Null for Collections

```java
// BAD: Returns null ❌
List<Order> getOrders(String userId) {
    if (noOrdersFound) {
        return null;  // Forces null check on caller
    }
}

// GOOD: Return empty collection ✅
List<Order> getOrders(String userId) {
    if (noOrdersFound) {
        return Collections.emptyList();
    }
}
```

### Use Optional for Nullable Returns

```java
// BAD: May return null ❌
User findByEmail(String email) {
    // Returns null if not found
}

// GOOD: Explicit optional ✅
Optional<User> findByEmail(String email) {
    User user = repository.find(email);
    return Optional.ofNullable(user);
}

// Usage
findByEmail("john@example.com")
    .ifPresent(user -> sendWelcomeEmail(user));

String name = findByEmail("john@example.com")
    .map(User::getName)
    .orElse("Guest");
```

---

## 4. Interface Design

### Program to Interface, Not Implementation

```java
// BAD: Coupled to implementation ❌
public class OrderService {
    private MySQLOrderRepository repository;  // Concrete class
}

// GOOD: Depends on interface ✅
public class OrderService {
    private OrderRepository repository;  // Interface
    
    public OrderService(OrderRepository repository) {
        this.repository = repository;
    }
}

interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(String id);
    List<Order> findByUserId(String userId);
}
```

### Interface Segregation

```java
// BAD: Fat interface ❌
interface DataProcessor {
    void read();
    void write();
    void compress();
    void encrypt();
    void validate();
}

// GOOD: Focused interfaces ✅
interface Readable { void read(); }
interface Writable { void write(); }
interface Compressible { void compress(); }
interface Encryptable { void encrypt(); }
interface Validatable { void validate(); }

// Implement only what you need
class FileProcessor implements Readable, Writable { }
```

---

## 5. Immutability

### Make Objects Immutable When Possible

```java
// Immutable class
public final class Money {
    private final double amount;
    private final String currency;
    
    public Money(double amount, String currency) {
        this.amount = amount;
        this.currency = currency;
    }
    
    // No setters, only getters
    public double getAmount() { return amount; }
    public String getCurrency() { return currency; }
    
    // Operations return new instances
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Currency mismatch");
        }
        return new Money(this.amount + other.amount, this.currency);
    }
    
    public Money multiply(double factor) {
        return new Money(this.amount * factor, this.currency);
    }
}

// Usage
Money price = new Money(100, "USD");
Money tax = new Money(10, "USD");
Money total = price.add(tax);  // New object, original unchanged
```

---

## 6. Defensive Copying

```java
public class Order {
    private final List<OrderItem> items;
    
    public Order(List<OrderItem> items) {
        // Defensive copy on input
        this.items = new ArrayList<>(items);
    }
    
    public List<OrderItem> getItems() {
        // Defensive copy on output
        return new ArrayList<>(items);
        // Or use unmodifiable:
        // return Collections.unmodifiableList(items);
    }
}
```

---

## 7. Input Validation

### Fail Fast with Validation

```java
public class OrderService {
    public Order createOrder(OrderRequest request) {
        // Validate ALL inputs at the start
        Objects.requireNonNull(request, "Request cannot be null");
        validateRequest(request);
        
        // Now proceed with business logic
        return processOrder(request);
    }
    
    private void validateRequest(OrderRequest request) {
        List<String> errors = new ArrayList<>();
        
        if (request.getItems() == null || request.getItems().isEmpty()) {
            errors.add("Order must have at least one item");
        }
        
        if (request.getShippingAddress() == null) {
            errors.add("Shipping address is required");
        }
        
        if (request.getPaymentMethod() == null) {
            errors.add("Payment method is required");
        }
        
        if (!errors.isEmpty()) {
            throw new ValidationException(errors);
        }
    }
}

// Validation exception
class ValidationException extends RuntimeException {
    private final List<String> errors;
    
    public ValidationException(List<String> errors) {
        super("Validation failed: " + String.join(", ", errors));
        this.errors = errors;
    }
    
    public List<String> getErrors() { return errors; }
}
```

---

## 8. Constants and Configuration

### Avoid Magic Numbers/Strings

```java
// BAD: Magic values ❌
if (user.getAge() >= 21) { }
if (status.equals("ACTIVE")) { }
Thread.sleep(60000);

// GOOD: Named constants ✅
private static final int LEGAL_DRINKING_AGE = 21;
private static final String STATUS_ACTIVE = "ACTIVE";
private static final long ONE_MINUTE_MS = 60_000;

if (user.getAge() >= LEGAL_DRINKING_AGE) { }
if (status.equals(STATUS_ACTIVE)) { }
Thread.sleep(ONE_MINUTE_MS);

// Even better: Use enums
enum UserStatus { ACTIVE, INACTIVE, SUSPENDED, DELETED }

if (user.getStatus() == UserStatus.ACTIVE) { }
```

---

## 9. Documentation

### Document Public APIs

```java
/**
 * Processes a payment for the given order.
 * 
 * <p>This method validates the payment details, charges the customer,
 * and creates a transaction record. If payment fails, the order status
 * is set to PAYMENT_FAILED.
 * 
 * @param order the order to process payment for (must not be null)
 * @param paymentDetails the payment information
 * @return PaymentResult containing transaction id if successful
 * @throws PaymentDeclinedException if the payment was declined
 * @throws ValidationException if order or payment details are invalid
 * @throws InsufficientFundsException if customer has insufficient balance
 */
public PaymentResult processPayment(Order order, PaymentDetails paymentDetails);
```

---

## 10. Versioning APIs

```java
// Version in package name
package com.company.api.v1;
package com.company.api.v2;

// Or version in interface/class name
interface PaymentService {
    PaymentResult processPaymentV1(Order order);  // Legacy
    PaymentResult processPaymentV2(PaymentRequest request);  // New
}

// Deprecated annotations
@Deprecated(since = "2.0", forRemoval = true)
public PaymentResult processPaymentV1(Order order) {
    // Legacy implementation
}
```

---

## API Design Checklist

| Check | Question |
|-------|----------|
| ✅ **Naming** | Is the method name clear and descriptive? |
| ✅ **Parameters** | Are there fewer than 4 parameters? |
| ✅ **Null Safety** | Are nulls handled properly? |
| ✅ **Return Types** | Are return types appropriate? |
| ✅ **Immutability** | Are objects immutable where possible? |
| ✅ **Validation** | Are inputs validated early? |
| ✅ **Documentation** | Is the public API documented? |
| ✅ **Consistency** | Does it follow existing patterns? |

---

**Next: Database Design →**
