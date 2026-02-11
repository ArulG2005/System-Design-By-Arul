# Facade Design Pattern

## Intent

> **Provide a unified interface to a set of interfaces in a subsystem. Facade defines a higher-level interface that makes the subsystem easier to use.**

---

## The Problem

You have a complex subsystem with many classes:
- Too many classes to interact with
- Complex dependencies between classes
- Clients need to know too much about the system
- Changes in subsystem affect all clients

---

## Simple Analogy

Think of a **Hotel Concierge**:
- You want: restaurant reservation, taxi, theater tickets
- Without concierge: call restaurant, call taxi company, call theater
- With concierge: "I need dinner and theater tonight" → all done!
- Concierge is the **facade** hiding complex bookings

Or **Starting a Modern Car**:
- Without facade: ignition, fuel pump, starter motor, battery, sensors...
- With facade: Just press the START button
- The button is the facade for the complex starting sequence

---

## Structure

```
┌─────────────────────────────────────────────────────┐
│                     Client                          │
└─────────────────────────┬───────────────────────────┘
                          │
                          │ simple interface
                          ▼
┌─────────────────────────────────────────────────────┐
│                     FACADE                          │
│  ┌─────────────────────────────────────────────┐   │
│  │ + simpleOperation()                          │   │
│  │ - subsystem1                                 │   │
│  │ - subsystem2                                 │   │
│  │ - subsystem3                                 │   │
│  └─────────────────────────────────────────────┘   │
└─────────────────────────┬───────────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          │               │               │
          ▼               ▼               ▼
    ┌───────────┐   ┌───────────┐   ┌───────────┐
    │Subsystem1 │   │Subsystem2 │   │Subsystem3 │
    ├───────────┤   ├───────────┤   ├───────────┤
    │+operation │   │+operation │   │+operation │
    └───────────┘   └───────────┘   └───────────┘
```

---

## Basic Example: Home Theater System

```java
// Complex subsystem classes
class DVDPlayer {
    public void on() {
        System.out.println("DVD Player ON");
    }
    
    public void off() {
        System.out.println("DVD Player OFF");
    }
    
    public void play(String movie) {
        System.out.println("Playing movie: " + movie);
    }
    
    public void stop() {
        System.out.println("DVD Player stopped");
    }
}

class Projector {
    public void on() {
        System.out.println("Projector ON");
    }
    
    public void off() {
        System.out.println("Projector OFF");
    }
    
    public void wideScreenMode() {
        System.out.println("Projector in widescreen mode");
    }
}

class SurroundSound {
    public void on() {
        System.out.println("Surround Sound ON");
    }
    
    public void off() {
        System.out.println("Surround Sound OFF");
    }
    
    public void setVolume(int level) {
        System.out.println("Volume set to " + level);
    }
}

class Lights {
    public void dim(int level) {
        System.out.println("Lights dimmed to " + level + "%");
    }
    
    public void on() {
        System.out.println("Lights ON");
    }
}

class PopcornMaker {
    public void on() {
        System.out.println("Popcorn Maker ON");
    }
    
    public void off() {
        System.out.println("Popcorn Maker OFF");
    }
    
    public void pop() {
        System.out.println("Making popcorn!");
    }
}

// FACADE - simplifies everything!
class HomeTheaterFacade {
    private DVDPlayer dvd;
    private Projector projector;
    private SurroundSound sound;
    private Lights lights;
    private PopcornMaker popcorn;
    
    public HomeTheaterFacade(DVDPlayer dvd, Projector projector,
                             SurroundSound sound, Lights lights,
                             PopcornMaker popcorn) {
        this.dvd = dvd;
        this.projector = projector;
        this.sound = sound;
        this.lights = lights;
        this.popcorn = popcorn;
    }
    
    // Simple method that handles all complexity
    public void watchMovie(String movie) {
        System.out.println("\n=== Getting ready to watch movie ===\n");
        popcorn.on();
        popcorn.pop();
        lights.dim(10);
        projector.on();
        projector.wideScreenMode();
        sound.on();
        sound.setVolume(50);
        dvd.on();
        dvd.play(movie);
    }
    
    // Simple method to end movie
    public void endMovie() {
        System.out.println("\n=== Shutting down movie theater ===\n");
        dvd.stop();
        dvd.off();
        sound.off();
        projector.off();
        lights.on();
        popcorn.off();
    }
}

// Client code
public class Main {
    public static void main(String[] args) {
        // Setup subsystems
        DVDPlayer dvd = new DVDPlayer();
        Projector projector = new Projector();
        SurroundSound sound = new SurroundSound();
        Lights lights = new Lights();
        PopcornMaker popcorn = new PopcornMaker();
        
        // Create facade
        HomeTheaterFacade theater = new HomeTheaterFacade(
            dvd, projector, sound, lights, popcorn);
        
        // Simple usage!
        theater.watchMovie("Inception");
        // ... enjoy movie ...
        theater.endMovie();
    }
}
```

**Without Facade**, client would need:
```java
// Lots of code every time!
popcorn.on();
popcorn.pop();
lights.dim(10);
projector.on();
projector.wideScreenMode();
sound.on();
sound.setVolume(50);
dvd.on();
dvd.play("Inception");
```

---

## Real-World Examples

### Example 1: E-Commerce Order Facade

```java
// Complex subsystem classes
class InventoryService {
    public boolean checkStock(String productId, int quantity) {
        System.out.println("Checking stock for " + productId);
        return true; // simplified
    }
    
    public void reduceStock(String productId, int quantity) {
        System.out.println("Reducing stock: " + productId + " by " + quantity);
    }
}

class PaymentService {
    public boolean validateCard(String cardNumber) {
        System.out.println("Validating card: ****" + cardNumber.substring(12));
        return true;
    }
    
    public boolean processPayment(String cardNumber, double amount) {
        System.out.println("Processing payment: $" + amount);
        return true;
    }
    
    public void refund(String transactionId) {
        System.out.println("Refunding transaction: " + transactionId);
    }
}

class ShippingService {
    public String calculateShipping(String address) {
        System.out.println("Calculating shipping to: " + address);
        return "5-7 business days";
    }
    
    public String createShipment(String orderId, String address) {
        System.out.println("Creating shipment for order: " + orderId);
        return "TRACK123456";
    }
}

class NotificationService {
    public void sendOrderConfirmation(String email, String orderId) {
        System.out.println("Sending confirmation to: " + email);
    }
    
    public void sendShippingNotification(String email, String trackingId) {
        System.out.println("Sending shipping notification: " + trackingId);
    }
}

class FraudDetectionService {
    public boolean checkFraud(String userId, double amount) {
        System.out.println("Checking fraud for user: " + userId);
        return false; // no fraud
    }
}

// Order data class
class Order {
    private String orderId;
    private String productId;
    private int quantity;
    private double amount;
    private String cardNumber;
    private String email;
    private String address;
    private String userId;
    
    // Constructor and getters
    public Order(String orderId, String productId, int quantity, 
                 double amount, String cardNumber, String email,
                 String address, String userId) {
        this.orderId = orderId;
        this.productId = productId;
        this.quantity = quantity;
        this.amount = amount;
        this.cardNumber = cardNumber;
        this.email = email;
        this.address = address;
        this.userId = userId;
    }
    
    // Getters
    public String getOrderId() { return orderId; }
    public String getProductId() { return productId; }
    public int getQuantity() { return quantity; }
    public double getAmount() { return amount; }
    public String getCardNumber() { return cardNumber; }
    public String getEmail() { return email; }
    public String getAddress() { return address; }
    public String getUserId() { return userId; }
}

// FACADE
class OrderFacade {
    private InventoryService inventory;
    private PaymentService payment;
    private ShippingService shipping;
    private NotificationService notification;
    private FraudDetectionService fraudDetection;
    
    public OrderFacade() {
        this.inventory = new InventoryService();
        this.payment = new PaymentService();
        this.shipping = new ShippingService();
        this.notification = new NotificationService();
        this.fraudDetection = new FraudDetectionService();
    }
    
    // ONE simple method to place entire order!
    public boolean placeOrder(Order order) {
        System.out.println("\n=== Processing Order: " + order.getOrderId() + " ===\n");
        
        // Step 1: Check inventory
        if (!inventory.checkStock(order.getProductId(), order.getQuantity())) {
            System.out.println("Order failed: Out of stock");
            return false;
        }
        
        // Step 2: Fraud check
        if (fraudDetection.checkFraud(order.getUserId(), order.getAmount())) {
            System.out.println("Order failed: Fraud detected");
            return false;
        }
        
        // Step 3: Validate payment
        if (!payment.validateCard(order.getCardNumber())) {
            System.out.println("Order failed: Invalid card");
            return false;
        }
        
        // Step 4: Process payment
        if (!payment.processPayment(order.getCardNumber(), order.getAmount())) {
            System.out.println("Order failed: Payment failed");
            return false;
        }
        
        // Step 5: Reduce inventory
        inventory.reduceStock(order.getProductId(), order.getQuantity());
        
        // Step 6: Create shipment
        String trackingId = shipping.createShipment(order.getOrderId(), order.getAddress());
        
        // Step 7: Send notifications
        notification.sendOrderConfirmation(order.getEmail(), order.getOrderId());
        notification.sendShippingNotification(order.getEmail(), trackingId);
        
        System.out.println("\n=== Order Placed Successfully! ===\n");
        return true;
    }
}

// Client code - super simple!
public class ECommerce {
    public static void main(String[] args) {
        OrderFacade orderFacade = new OrderFacade();
        
        Order order = new Order(
            "ORD001", "PROD123", 2, 
            99.99, "1234567890123456",
            "user@email.com", "123 Main St",
            "USER001");
        
        boolean success = orderFacade.placeOrder(order);
    }
}
```

---

### Example 2: Computer Startup Facade

```java
// Complex subsystem
class CPU {
    public void freeze() {
        System.out.println("CPU: Freezing...");
    }
    
    public void jump(long position) {
        System.out.println("CPU: Jumping to " + position);
    }
    
    public void execute() {
        System.out.println("CPU: Executing...");
    }
}

class Memory {
    public void load(long position, byte[] data) {
        System.out.println("Memory: Loading data at " + position);
    }
}

class HardDrive {
    public byte[] read(long lba, int size) {
        System.out.println("HardDrive: Reading sector " + lba);
        return new byte[size];
    }
}

class GraphicsCard {
    public void initialize() {
        System.out.println("Graphics: Initializing...");
    }
    
    public void displayBootScreen() {
        System.out.println("Graphics: Displaying boot screen");
    }
}

class NetworkCard {
    public void initialize() {
        System.out.println("Network: Initializing...");
    }
    
    public void connectToNetwork() {
        System.out.println("Network: Connecting...");
    }
}

// FACADE
class ComputerFacade {
    private CPU cpu;
    private Memory memory;
    private HardDrive hardDrive;
    private GraphicsCard graphics;
    private NetworkCard network;
    
    private static final long BOOT_ADDRESS = 0x0000FFFF;
    private static final long BOOT_SECTOR = 0;
    private static final int SECTOR_SIZE = 512;
    
    public ComputerFacade() {
        this.cpu = new CPU();
        this.memory = new Memory();
        this.hardDrive = new HardDrive();
        this.graphics = new GraphicsCard();
        this.network = new NetworkCard();
    }
    
    // Simple start method
    public void start() {
        System.out.println("\n=== Starting Computer ===\n");
        
        cpu.freeze();
        graphics.initialize();
        graphics.displayBootScreen();
        
        byte[] bootData = hardDrive.read(BOOT_SECTOR, SECTOR_SIZE);
        memory.load(BOOT_ADDRESS, bootData);
        
        cpu.jump(BOOT_ADDRESS);
        cpu.execute();
        
        network.initialize();
        network.connectToNetwork();
        
        System.out.println("\n=== Computer Started! ===\n");
    }
    
    // Simple shutdown method
    public void shutdown() {
        System.out.println("\n=== Shutting Down ===\n");
        System.out.println("Saving state...");
        System.out.println("Closing applications...");
        System.out.println("Powering off...");
        System.out.println("\n=== Goodbye! ===");
    }
}

// Usage
public class Main {
    public static void main(String[] args) {
        ComputerFacade computer = new ComputerFacade();
        computer.start();  // Just one call!
        // ... use computer ...
        computer.shutdown();
    }
}
```

---

### Example 3: Banking Operations Facade

```java
// Subsystems
class AccountService {
    private Map<String, Double> balances = new HashMap<>();
    
    public AccountService() {
        balances.put("ACC001", 1000.0);
        balances.put("ACC002", 2000.0);
    }
    
    public double getBalance(String accountId) {
        return balances.getOrDefault(accountId, 0.0);
    }
    
    public void debit(String accountId, double amount) {
        balances.put(accountId, getBalance(accountId) - amount);
    }
    
    public void credit(String accountId, double amount) {
        balances.put(accountId, getBalance(accountId) + amount);
    }
}

class AuthenticationService {
    public boolean authenticate(String userId, String pin) {
        System.out.println("Authenticating user: " + userId);
        return pin.equals("1234"); // simplified
    }
}

class TransactionLogger {
    public void log(String transactionId, String details) {
        System.out.println("LOG [" + transactionId + "]: " + details);
    }
}

class FeeCalculator {
    public double calculateTransferFee(double amount) {
        return amount * 0.01; // 1% fee
    }
}

class NotificationService {
    public void notifyUser(String userId, String message) {
        System.out.println("NOTIFICATION to " + userId + ": " + message);
    }
}

// FACADE
class BankingFacade {
    private AccountService accounts;
    private AuthenticationService auth;
    private TransactionLogger logger;
    private FeeCalculator feeCalculator;
    private NotificationService notifications;
    
    public BankingFacade() {
        this.accounts = new AccountService();
        this.auth = new AuthenticationService();
        this.logger = new TransactionLogger();
        this.feeCalculator = new FeeCalculator();
        this.notifications = new NotificationService();
    }
    
    // Simple transfer method
    public boolean transfer(String userId, String pin, 
                           String fromAccount, String toAccount, 
                           double amount) {
        String txnId = "TXN" + System.currentTimeMillis();
        
        // Authenticate
        if (!auth.authenticate(userId, pin)) {
            logger.log(txnId, "Authentication failed");
            return false;
        }
        
        // Check balance
        double balance = accounts.getBalance(fromAccount);
        double fee = feeCalculator.calculateTransferFee(amount);
        double total = amount + fee;
        
        if (balance < total) {
            logger.log(txnId, "Insufficient funds");
            return false;
        }
        
        // Perform transfer
        accounts.debit(fromAccount, total);
        accounts.credit(toAccount, amount);
        
        // Log and notify
        logger.log(txnId, "Transferred " + amount + " from " + 
                   fromAccount + " to " + toAccount);
        notifications.notifyUser(userId, "Transfer of $" + amount + " completed");
        
        return true;
    }
    
    // Simple balance check
    public double checkBalance(String userId, String pin, String accountId) {
        if (!auth.authenticate(userId, pin)) {
            throw new RuntimeException("Authentication failed");
        }
        return accounts.getBalance(accountId);
    }
}

// Usage
public class BankApp {
    public static void main(String[] args) {
        BankingFacade bank = new BankingFacade();
        
        // Simple call to transfer money
        boolean success = bank.transfer(
            "USER001", "1234",
            "ACC001", "ACC002",
            100.0);
        
        // Check balance
        double balance = bank.checkBalance("USER001", "1234", "ACC001");
        System.out.println("Balance: $" + balance);
    }
}
```

---

## When to Use Facade Pattern

### ✅ Use When:
1. **Complex subsystem** - Too many classes to work with
2. **Hide complexity** - Clients don't need to know internals
3. **Reduce dependencies** - Minimize client-subsystem coupling
4. **Layer your system** - Create entry points to each layer

### ❌ Don't Use When:
1. Subsystem is simple enough
2. Clients need fine-grained control
3. You want direct access to subsystem

---

## Facade vs Other Patterns

| Pattern | Purpose |
|---------|---------|
| **Facade** | Simplify interface to complex subsystem |
| **Adapter** | Convert one interface to another |
| **Mediator** | Coordinate between colleague objects |
| **Proxy** | Control access to object |

---

## Key Points

1. **Facade doesn't hide subsystem** - Clients can still access it directly if needed
2. **Multiple facades** - You can have different facades for different clients
3. **One-way simplification** - Facade knows subsystem, subsystem doesn't know facade

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Intent** | Provide simple interface to complex subsystem |
| **Key Idea** | One class that coordinates multiple subsystem classes |
| **Benefits** | Reduces complexity, decouples client from subsystem |
| **Use Case** | Complex libraries, legacy systems, layered architecture |

### Remember:
- Facade = **Simplified interface**
- Client → Facade → Subsystems
- Subsystems stay accessible
- Good for **third-party libraries** and **legacy code**

---

**Next: Composite Pattern →**
