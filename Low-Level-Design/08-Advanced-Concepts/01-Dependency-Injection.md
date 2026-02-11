# Dependency Injection (DI)

## What is Dependency Injection?

> **Dependency Injection is a design pattern where an object receives its dependencies from external sources rather than creating them itself.**

---

## The Problem Without DI

```java
// Tightly coupled - BAD! ❌
class OrderService {
    private MySQLDatabase database;  // Hardcoded dependency
    private EmailService emailService;
    
    public OrderService() {
        this.database = new MySQLDatabase();  // Creates its own dependency
        this.emailService = new EmailService();
    }
    
    public void createOrder(Order order) {
        database.save(order);
        emailService.send("Order created!");
    }
}

// Problems:
// 1. Cannot swap MySQL for PostgreSQL easily
// 2. Cannot mock database for testing
// 3. OrderService knows TOO MUCH about dependencies
// 4. Violates Single Responsibility Principle
```

---

## The Solution: Dependency Injection

```java
// Loosely coupled - GOOD! ✅
class OrderService {
    private Database database;        // Interface, not concrete class
    private NotificationService notifier;
    
    // Dependencies INJECTED from outside
    public OrderService(Database database, NotificationService notifier) {
        this.database = database;
        this.notifier = notifier;
    }
    
    public void createOrder(Order order) {
        database.save(order);
        notifier.send("Order created!");
    }
}

// Now we can inject any implementation!
Database mysql = new MySQLDatabase();
Database postgres = new PostgreSQLDatabase();
Database mock = new MockDatabase();  // For testing!

OrderService service = new OrderService(mysql, new EmailService());
```

---

## Types of Dependency Injection

### 1. Constructor Injection (Recommended)

```java
class UserService {
    private final UserRepository repository;
    private final EmailService emailService;
    
    // Dependencies provided through constructor
    public UserService(UserRepository repository, EmailService emailService) {
        this.repository = repository;
        this.emailService = emailService;
    }
}

// Usage
UserService service = new UserService(
    new MySQLUserRepository(),
    new SMTPEmailService()
);
```

**Pros:**
- Dependencies are clear and explicit
- Object is always in valid state
- Can use `final` fields (immutable)
- Easy to test

---

### 2. Setter Injection

```java
class ReportGenerator {
    private DataSource dataSource;
    private Formatter formatter;
    
    // Dependencies provided through setters
    public void setDataSource(DataSource dataSource) {
        this.dataSource = dataSource;
    }
    
    public void setFormatter(Formatter formatter) {
        this.formatter = formatter;
    }
}

// Usage
ReportGenerator generator = new ReportGenerator();
generator.setDataSource(new DatabaseSource());
generator.setFormatter(new PDFFormatter());
```

**Pros:**
- Optional dependencies
- Can change dependencies at runtime

**Cons:**
- Object may be in invalid state
- Dependencies not immediately visible

---

### 3. Interface Injection

```java
// Interface defines injection method
interface DatabaseInjector {
    void injectDatabase(Database database);
}

class ProductService implements DatabaseInjector {
    private Database database;
    
    @Override
    public void injectDatabase(Database database) {
        this.database = database;
    }
}
```

---

## Complete Example: E-Commerce System

```java
// Interfaces (abstractions)
interface PaymentGateway {
    boolean processPayment(double amount);
}

interface InventoryService {
    boolean checkStock(String productId);
    void reduceStock(String productId, int quantity);
}

interface NotificationService {
    void sendNotification(String userId, String message);
}

interface OrderRepository {
    void save(Order order);
    Order findById(String orderId);
}

// Concrete implementations
class StripePaymentGateway implements PaymentGateway {
    @Override
    public boolean processPayment(double amount) {
        System.out.println("Processing $" + amount + " via Stripe");
        return true;
    }
}

class PayPalGateway implements PaymentGateway {
    @Override
    public boolean processPayment(double amount) {
        System.out.println("Processing $" + amount + " via PayPal");
        return true;
    }
}

class WarehouseInventoryService implements InventoryService {
    @Override
    public boolean checkStock(String productId) {
        System.out.println("Checking stock for " + productId);
        return true;
    }
    
    @Override
    public void reduceStock(String productId, int quantity) {
        System.out.println("Reducing stock: " + productId + " by " + quantity);
    }
}

class EmailNotificationService implements NotificationService {
    @Override
    public void sendNotification(String userId, String message) {
        System.out.println("Email to " + userId + ": " + message);
    }
}

class MySQLOrderRepository implements OrderRepository {
    @Override
    public void save(Order order) {
        System.out.println("Saving order to MySQL");
    }
    
    @Override
    public Order findById(String orderId) {
        return new Order(orderId);
    }
}

// Order class
class Order {
    private String orderId;
    private String userId;
    private String productId;
    private int quantity;
    private double amount;
    
    public Order(String orderId) {
        this.orderId = orderId;
    }
    
    // Getters and setters
    public String getOrderId() { return orderId; }
    public String getUserId() { return userId; }
    public String getProductId() { return productId; }
    public int getQuantity() { return quantity; }
    public double getAmount() { return amount; }
    
    public void setUserId(String userId) { this.userId = userId; }
    public void setProductId(String productId) { this.productId = productId; }
    public void setQuantity(int quantity) { this.quantity = quantity; }
    public void setAmount(double amount) { this.amount = amount; }
}

// Service with injected dependencies
class OrderService {
    private final PaymentGateway paymentGateway;
    private final InventoryService inventoryService;
    private final NotificationService notificationService;
    private final OrderRepository orderRepository;
    
    // All dependencies injected through constructor
    public OrderService(
            PaymentGateway paymentGateway,
            InventoryService inventoryService,
            NotificationService notificationService,
            OrderRepository orderRepository) {
        this.paymentGateway = paymentGateway;
        this.inventoryService = inventoryService;
        this.notificationService = notificationService;
        this.orderRepository = orderRepository;
    }
    
    public boolean placeOrder(Order order) {
        // Check inventory
        if (!inventoryService.checkStock(order.getProductId())) {
            notificationService.sendNotification(
                order.getUserId(), "Sorry, item out of stock!");
            return false;
        }
        
        // Process payment
        if (!paymentGateway.processPayment(order.getAmount())) {
            notificationService.sendNotification(
                order.getUserId(), "Payment failed!");
            return false;
        }
        
        // Reduce inventory
        inventoryService.reduceStock(order.getProductId(), order.getQuantity());
        
        // Save order
        orderRepository.save(order);
        
        // Notify user
        notificationService.sendNotification(
            order.getUserId(), "Order placed successfully!");
        
        return true;
    }
}

// Application - wiring dependencies
public class Application {
    public static void main(String[] args) {
        // Create dependencies
        PaymentGateway payment = new StripePaymentGateway();
        InventoryService inventory = new WarehouseInventoryService();
        NotificationService notification = new EmailNotificationService();
        OrderRepository repository = new MySQLOrderRepository();
        
        // Inject into service
        OrderService orderService = new OrderService(
            payment, inventory, notification, repository
        );
        
        // Use service
        Order order = new Order("ORD-001");
        order.setUserId("user123");
        order.setProductId("PROD-001");
        order.setQuantity(2);
        order.setAmount(99.99);
        
        orderService.placeOrder(order);
    }
}
```

---

## Testing with DI

```java
// Mock implementations for testing
class MockPaymentGateway implements PaymentGateway {
    public boolean shouldSucceed = true;
    public int callCount = 0;
    
    @Override
    public boolean processPayment(double amount) {
        callCount++;
        return shouldSucceed;
    }
}

class MockInventoryService implements InventoryService {
    public boolean hasStock = true;
    
    @Override
    public boolean checkStock(String productId) {
        return hasStock;
    }
    
    @Override
    public void reduceStock(String productId, int quantity) {
        // Do nothing in mock
    }
}

class MockNotificationService implements NotificationService {
    public List<String> sentMessages = new ArrayList<>();
    
    @Override
    public void sendNotification(String userId, String message) {
        sentMessages.add(message);
    }
}

class MockOrderRepository implements OrderRepository {
    public List<Order> savedOrders = new ArrayList<>();
    
    @Override
    public void save(Order order) {
        savedOrders.add(order);
    }
    
    @Override
    public Order findById(String orderId) {
        return savedOrders.stream()
            .filter(o -> o.getOrderId().equals(orderId))
            .findFirst()
            .orElse(null);
    }
}

// Unit Test
class OrderServiceTest {
    private MockPaymentGateway mockPayment;
    private MockInventoryService mockInventory;
    private MockNotificationService mockNotification;
    private MockOrderRepository mockRepository;
    private OrderService orderService;
    
    @Before
    public void setUp() {
        mockPayment = new MockPaymentGateway();
        mockInventory = new MockInventoryService();
        mockNotification = new MockNotificationService();
        mockRepository = new MockOrderRepository();
        
        // Inject mocks!
        orderService = new OrderService(
            mockPayment, mockInventory, mockNotification, mockRepository
        );
    }
    
    @Test
    public void testSuccessfulOrder() {
        Order order = createTestOrder();
        
        boolean result = orderService.placeOrder(order);
        
        assertTrue(result);
        assertEquals(1, mockPayment.callCount);
        assertEquals(1, mockRepository.savedOrders.size());
        assertTrue(mockNotification.sentMessages.get(0).contains("successfully"));
    }
    
    @Test
    public void testOutOfStock() {
        mockInventory.hasStock = false;  // Simulate out of stock
        Order order = createTestOrder();
        
        boolean result = orderService.placeOrder(order);
        
        assertFalse(result);
        assertEquals(0, mockPayment.callCount);  // Payment not called
        assertTrue(mockNotification.sentMessages.get(0).contains("out of stock"));
    }
    
    @Test
    public void testPaymentFailure() {
        mockPayment.shouldSucceed = false;  // Simulate payment failure
        Order order = createTestOrder();
        
        boolean result = orderService.placeOrder(order);
        
        assertFalse(result);
        assertEquals(0, mockRepository.savedOrders.size());  // Order not saved
    }
    
    private Order createTestOrder() {
        Order order = new Order("TEST-001");
        order.setUserId("testUser");
        order.setProductId("PROD-001");
        order.setQuantity(1);
        order.setAmount(50.0);
        return order;
    }
}
```

---

## Simple DI Container

```java
// Simple DI Container (like Spring, but simpler)
class DIContainer {
    private Map<Class<?>, Object> instances = new HashMap<>();
    private Map<Class<?>, Class<?>> bindings = new HashMap<>();
    
    // Bind interface to implementation
    public <T> void bind(Class<T> interfaceClass, Class<? extends T> implClass) {
        bindings.put(interfaceClass, implClass);
    }
    
    // Register singleton instance
    public <T> void singleton(Class<T> clazz, T instance) {
        instances.put(clazz, instance);
    }
    
    // Resolve dependency
    @SuppressWarnings("unchecked")
    public <T> T resolve(Class<T> clazz) {
        // Check if singleton exists
        if (instances.containsKey(clazz)) {
            return (T) instances.get(clazz);
        }
        
        // Check if binding exists
        Class<?> implClass = bindings.getOrDefault(clazz, clazz);
        
        try {
            // Get constructor
            Constructor<?> constructor = implClass.getConstructors()[0];
            
            // Resolve constructor parameters
            Class<?>[] paramTypes = constructor.getParameterTypes();
            Object[] params = new Object[paramTypes.length];
            
            for (int i = 0; i < paramTypes.length; i++) {
                params[i] = resolve(paramTypes[i]);  // Recursive resolution
            }
            
            // Create instance
            T instance = (T) constructor.newInstance(params);
            return instance;
            
        } catch (Exception e) {
            throw new RuntimeException("Failed to resolve: " + clazz, e);
        }
    }
}

// Usage
public class ContainerDemo {
    public static void main(String[] args) {
        DIContainer container = new DIContainer();
        
        // Configure bindings
        container.bind(PaymentGateway.class, StripePaymentGateway.class);
        container.bind(InventoryService.class, WarehouseInventoryService.class);
        container.bind(NotificationService.class, EmailNotificationService.class);
        container.bind(OrderRepository.class, MySQLOrderRepository.class);
        
        // Container automatically resolves all dependencies!
        OrderService orderService = container.resolve(OrderService.class);
        
        // Use the service
        Order order = new Order("ORD-001");
        orderService.placeOrder(order);
    }
}
```

---

## Benefits of Dependency Injection

| Benefit | Description |
|---------|-------------|
| **Loose Coupling** | Classes don't depend on concrete implementations |
| **Testability** | Easy to inject mocks for testing |
| **Flexibility** | Swap implementations without changing code |
| **Single Responsibility** | Objects don't create their dependencies |
| **Reusability** | Components can be reused in different contexts |

---

## DI Frameworks

| Framework | Language | Notes |
|-----------|----------|-------|
| **Spring** | Java | Most popular, full-featured |
| **Guice** | Java | Lightweight, by Google |
| **Dagger** | Java/Android | Compile-time DI |
| **CDI** | Java EE | Standard for Java EE |

---

## Summary

### Without DI:
```java
class Service {
    private Database db = new MySQL();  // Tightly coupled
}
```

### With DI:
```java
class Service {
    private Database db;
    public Service(Database db) { this.db = db; }  // Loosely coupled
}
```

### Remember:
- **Depend on abstractions**, not concretions
- **Inject dependencies**, don't create them
- **Constructor injection** is preferred
- DI enables **easy testing**

---

**Next: Exception Handling and Resilience →**
