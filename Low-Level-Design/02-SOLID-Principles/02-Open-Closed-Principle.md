# Open/Closed Principle (OCP)

## The Second SOLID Principle

> **"Software entities (classes, modules, functions) should be open for extension, but closed for modification."**
> — Bertrand Meyer

---

## What Does This Mean?

- **Open for Extension**: You can ADD new functionality
- **Closed for Modification**: You should NOT CHANGE existing code

In simple words: **Add new features by writing NEW code, not changing OLD code.**

---

## Why is OCP Important?

When you modify existing code:
- You might break things that were working
- You need to re-test everything
- Other parts depending on this code might fail

When you extend with new code:
- Existing functionality remains untouched
- Fewer bugs in production
- Easier to add new features

---

## Simple Analogy

Think of a **smartphone app store**:
- The phone OS is **closed for modification** (Apple/Google controls it)
- But it's **open for extension** (anyone can add new apps)

You don't modify the OS to add a new game - you simply install the app!

---

## Real-World Example: Calculating Area

### ❌ BAD: Violating OCP

```java
class Rectangle {
    public double width;
    public double height;
}

class Circle {
    public double radius;
}

class AreaCalculator {
    public double calculateArea(Object shape) {
        if (shape instanceof Rectangle) {
            Rectangle rectangle = (Rectangle) shape;
            return rectangle.width * rectangle.height;
        } 
        else if (shape instanceof Circle) {
            Circle circle = (Circle) shape;
            return Math.PI * circle.radius * circle.radius;
        }
        // What if we add Triangle? We need to MODIFY this class!
        // What if we add Pentagon? We need to MODIFY again!
        else if (shape instanceof Triangle) {
            Triangle triangle = (Triangle) shape;
            return 0.5 * triangle.base * triangle.height;
        }
        // This will keep growing forever!
        return 0;
    }
}
```

### Problems:
1. Every new shape requires modifying `AreaCalculator`
2. The `if-else` chain grows endlessly
3. High risk of breaking existing calculations
4. Violates SRP too (calculates for ALL shapes)

---

### ✅ GOOD: Following OCP

```java
// Step 1: Create an abstraction (interface)
interface Shape {
    double calculateArea();
}

// Step 2: Each shape implements the interface
class Rectangle implements Shape {
    private double width;
    private double height;
    
    public Rectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }
    
    @Override
    public double calculateArea() {
        return width * height;
    }
}

class Circle implements Shape {
    private double radius;
    
    public Circle(double radius) {
        this.radius = radius;
    }
    
    @Override
    public double calculateArea() {
        return Math.PI * radius * radius;
    }
}

// Step 3: AreaCalculator works with the interface
class AreaCalculator {
    public double calculateTotalArea(List<Shape> shapes) {
        double total = 0;
        for (Shape shape : shapes) {
            total += shape.calculateArea();  // Works for ANY shape!
        }
        return total;
    }
}
```

### Adding New Shapes (Without Modification):

```java
// Just add NEW classes - no modification to existing code!
class Triangle implements Shape {
    private double base;
    private double height;
    
    public Triangle(double base, double height) {
        this.base = base;
        this.height = height;
    }
    
    @Override
    public double calculateArea() {
        return 0.5 * base * height;
    }
}

class Pentagon implements Shape {
    private double side;
    
    public Pentagon(double side) {
        this.side = side;
    }
    
    @Override
    public double calculateArea() {
        return 0.25 * Math.sqrt(5 * (5 + 2 * Math.sqrt(5))) * side * side;
    }
}

// AreaCalculator works WITHOUT ANY CHANGES!
List<Shape> shapes = Arrays.asList(
    new Rectangle(10, 5),
    new Circle(7),
    new Triangle(4, 3),
    new Pentagon(6)
);

AreaCalculator calculator = new AreaCalculator();
double total = calculator.calculateTotalArea(shapes);  // Just works!
```

---

## Another Example: Payment Processing

### ❌ BAD: Violating OCP

```java
class PaymentProcessor {
    public void processPayment(String paymentType, double amount) {
        if (paymentType.equals("creditCard")) {
            System.out.println("Processing credit card payment: $" + amount);
            // Credit card specific logic
        } 
        else if (paymentType.equals("paypal")) {
            System.out.println("Processing PayPal payment: $" + amount);
            // PayPal specific logic
        }
        else if (paymentType.equals("bitcoin")) {
            // Added later - had to MODIFY this class!
            System.out.println("Processing Bitcoin payment: $" + amount);
        }
        else if (paymentType.equals("applePay")) {
            // Added even later - MODIFIED again!
            System.out.println("Processing Apple Pay payment: $" + amount);
        }
        // This keeps growing...
    }
}
```

---

### ✅ GOOD: Following OCP

```java
// Step 1: Define an interface
interface PaymentMethod {
    void processPayment(double amount);
    String getPaymentType();
}

// Step 2: Implement for each payment method
class CreditCardPayment implements PaymentMethod {
    private String cardNumber;
    
    public CreditCardPayment(String cardNumber) {
        this.cardNumber = cardNumber;
    }
    
    @Override
    public void processPayment(double amount) {
        System.out.println("Processing credit card payment: $" + amount);
        System.out.println("Card: " + maskCardNumber());
        // Credit card specific logic: connect to bank, validate, etc.
    }
    
    @Override
    public String getPaymentType() {
        return "Credit Card";
    }
    
    private String maskCardNumber() {
        return "****-****-****-" + cardNumber.substring(cardNumber.length() - 4);
    }
}

class PayPalPayment implements PaymentMethod {
    private String email;
    
    public PayPalPayment(String email) {
        this.email = email;
    }
    
    @Override
    public void processPayment(double amount) {
        System.out.println("Processing PayPal payment: $" + amount);
        System.out.println("PayPal account: " + email);
        // PayPal specific logic: redirect to PayPal, etc.
    }
    
    @Override
    public String getPaymentType() {
        return "PayPal";
    }
}

// Step 3: PaymentProcessor is now closed for modification
class PaymentProcessor {
    public void processPayment(PaymentMethod paymentMethod, double amount) {
        System.out.println("Starting " + paymentMethod.getPaymentType() + " payment...");
        paymentMethod.processPayment(amount);
        System.out.println("Payment complete!");
    }
}
```

### Adding New Payment Methods (No Modification):

```java
// Add Bitcoin - no changes to PaymentProcessor!
class BitcoinPayment implements PaymentMethod {
    private String walletAddress;
    
    public BitcoinPayment(String walletAddress) {
        this.walletAddress = walletAddress;
    }
    
    @Override
    public void processPayment(double amount) {
        double btcAmount = amount / 50000; // Example conversion
        System.out.println("Processing Bitcoin payment: " + btcAmount + " BTC");
        System.out.println("Wallet: " + walletAddress);
    }
    
    @Override
    public String getPaymentType() {
        return "Bitcoin";
    }
}

// Add UPI - no changes to PaymentProcessor!
class UPIPayment implements PaymentMethod {
    private String upiId;
    
    public UPIPayment(String upiId) {
        this.upiId = upiId;
    }
    
    @Override
    public void processPayment(double amount) {
        System.out.println("Processing UPI payment: ₹" + amount);
        System.out.println("UPI ID: " + upiId);
    }
    
    @Override
    public String getPaymentType() {
        return "UPI";
    }
}

// Usage
PaymentProcessor processor = new PaymentProcessor();
processor.processPayment(new CreditCardPayment("1234567890123456"), 100.0);
processor.processPayment(new PayPalPayment("user@email.com"), 50.0);
processor.processPayment(new BitcoinPayment("1A2B3C4D5E"), 200.0);
processor.processPayment(new UPIPayment("user@okicici"), 500.0);
```

---

## Another Example: Notification System

### ❌ BAD: Violating OCP

```java
class NotificationService {
    public void send(String type, String message, String recipient) {
        if (type.equals("email")) {
            // Email logic
            System.out.println("Sending email to " + recipient + ": " + message);
        } else if (type.equals("sms")) {
            // SMS logic
            System.out.println("Sending SMS to " + recipient + ": " + message);
        } else if (type.equals("push")) {
            // Push notification logic
            System.out.println("Sending push to " + recipient + ": " + message);
        }
        // Add WhatsApp? Modify this class!
        // Add Slack? Modify again!
    }
}
```

### ✅ GOOD: Following OCP

```java
interface NotificationChannel {
    void send(String message, String recipient);
}

class EmailNotification implements NotificationChannel {
    @Override
    public void send(String message, String recipient) {
        System.out.println("📧 Email to " + recipient + ": " + message);
        // SMTP logic
    }
}

class SMSNotification implements NotificationChannel {
    @Override
    public void send(String message, String recipient) {
        System.out.println("📱 SMS to " + recipient + ": " + message);
        // SMS gateway logic
    }
}

class PushNotification implements NotificationChannel {
    @Override
    public void send(String message, String recipient) {
        System.out.println("🔔 Push to " + recipient + ": " + message);
        // Firebase/APNs logic
    }
}

// Adding new channels is easy!
class WhatsAppNotification implements NotificationChannel {
    @Override
    public void send(String message, String recipient) {
        System.out.println("💬 WhatsApp to " + recipient + ": " + message);
    }
}

class SlackNotification implements NotificationChannel {
    @Override
    public void send(String message, String recipient) {
        System.out.println("🔷 Slack to " + recipient + ": " + message);
    }
}

// NotificationService is closed for modification
class NotificationService {
    private List<NotificationChannel> channels;
    
    public NotificationService(List<NotificationChannel> channels) {
        this.channels = channels;
    }
    
    public void notifyAll(String message, String recipient) {
        for (NotificationChannel channel : channels) {
            channel.send(message, recipient);
        }
    }
}
```

---

## Techniques to Achieve OCP

### 1. **Strategy Pattern**
Use different algorithms interchangeably.

```java
interface SortingStrategy {
    void sort(int[] arr);
}

class BubbleSort implements SortingStrategy { /* ... */ }
class QuickSort implements SortingStrategy { /* ... */ }
class MergeSort implements SortingStrategy { /* ... */ }

class Sorter {
    private SortingStrategy strategy;
    
    public void setStrategy(SortingStrategy strategy) {
        this.strategy = strategy;
    }
    
    public void sort(int[] arr) {
        strategy.sort(arr);
    }
}
```

### 2. **Template Method Pattern**
Define skeleton in base class, let subclasses fill in details.

```java
abstract class DataProcessor {
    // Template method - closed for modification
    public final void process() {
        readData();
        processData();
        writeData();
    }
    
    protected abstract void readData();    // Open for extension
    protected abstract void processData(); // Open for extension
    protected abstract void writeData();   // Open for extension
}

class CSVProcessor extends DataProcessor {
    @Override
    protected void readData() { /* CSV reading */ }
    
    @Override
    protected void processData() { /* CSV processing */ }
    
    @Override
    protected void writeData() { /* CSV writing */ }
}

class JSONProcessor extends DataProcessor {
    @Override
    protected void readData() { /* JSON reading */ }
    
    @Override
    protected void processData() { /* JSON processing */ }
    
    @Override
    protected void writeData() { /* JSON writing */ }
}
```

### 3. **Decorator Pattern**
Add behavior without modifying existing code.

```java
interface Coffee {
    double getCost();
    String getDescription();
}

class SimpleCoffee implements Coffee {
    public double getCost() { return 2.0; }
    public String getDescription() { return "Coffee"; }
}

// Decorators extend functionality without modifying SimpleCoffee
class MilkDecorator implements Coffee {
    private Coffee coffee;
    
    public MilkDecorator(Coffee coffee) {
        this.coffee = coffee;
    }
    
    public double getCost() { return coffee.getCost() + 0.5; }
    public String getDescription() { return coffee.getDescription() + ", Milk"; }
}

class SugarDecorator implements Coffee {
    private Coffee coffee;
    
    public SugarDecorator(Coffee coffee) {
        this.coffee = coffee;
    }
    
    public double getCost() { return coffee.getCost() + 0.2; }
    public String getDescription() { return coffee.getDescription() + ", Sugar"; }
}
```

---

## Identifying OCP Violations

### Red Flags 🚩

1. **Switch statements on type**
```java
switch (type) {
    case "A": doA(); break;
    case "B": doB(); break;
    // Adding new type = modifying this
}
```

2. **instanceof checks**
```java
if (obj instanceof TypeA) { }
else if (obj instanceof TypeB) { }
// Adding new type = modifying this
```

3. **Frequent modifications to the same class**
```java
// Git history shows:
// "Added support for PDF"
// "Added support for Excel"
// "Added support for Word"
// Same class modified repeatedly!
```

4. **Large if-else chains based on type**

---

## When to Apply OCP

### Apply OCP When:
- You expect NEW variations (payment types, file formats, etc.)
- The class is modified frequently for similar reasons
- Multiple developers might add similar features

### Maybe Skip OCP When:
- The code is simple and unlikely to change
- Adding abstraction creates unnecessary complexity
- You're prototyping (can refactor later)

---

## OCP in Real Frameworks

### Java Collections Framework
```java
// List interface is closed
// But open for extension: ArrayList, LinkedList, Vector, Stack...

List<String> list1 = new ArrayList<>();
List<String> list2 = new LinkedList<>();
List<String> list3 = new Vector<>();
```

### Java I/O Streams
```java
// InputStream is abstract, closed for modification
// But open for extension: FileInputStream, ByteArrayInputStream, etc.

InputStream stream1 = new FileInputStream("file.txt");
InputStream stream2 = new ByteArrayInputStream(bytes);
InputStream stream3 = new BufferedInputStream(stream1);
```

### Spring Framework
```java
// You extend Spring by implementing interfaces
// Not by modifying Spring code

@Controller  // You implement controller
@Service     // You implement service
@Repository  // You implement repository
```

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Principle** | Open for extension, closed for modification |
| **Key Technique** | Use interfaces/abstract classes |
| **How to Extend** | Create new classes that implement the interface |
| **Benefit** | Add features without breaking existing code |
| **Patterns** | Strategy, Template Method, Decorator |

### Remember:
- Use **interfaces** to define contracts
- **New features** = New classes implementing the interface
- **Existing code** remains untouched
- Think: "How can I add this without changing existing code?"

---

**Next: Liskov Substitution Principle (LSP) →**
