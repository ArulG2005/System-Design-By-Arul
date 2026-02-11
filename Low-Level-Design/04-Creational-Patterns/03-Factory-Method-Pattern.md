# Factory Method Design Pattern

## Intent

> **Define an interface for creating objects, but let subclasses decide which class to instantiate.**

The Factory Method lets a class defer instantiation to subclasses.

---

## The Problem

You need to create objects, but:
- You don't know the exact type until runtime
- Adding new types shouldn't require modifying existing code
- Object creation logic is complex and shouldn't be duplicated

---

## Simple Analogy

Think of a **Pizza Store franchise**:
- Headquarters defines HOW to make pizza (process)
- Each store DECIDES which pizza to make (New York style, Chicago style)
- Customers just order "pizza" - they get the local style

The "create pizza" method is the Factory Method.

---

## Types of Factory Patterns

| Pattern | Description |
|---------|-------------|
| **Simple Factory** | One class creates objects (not GoF pattern) |
| **Factory Method** | Subclasses decide what to create |
| **Abstract Factory** | Create families of related objects |

---

## Simple Factory (Not a GoF Pattern, but Common)

A single class with a method that creates objects.

```java
// Products
interface Notification {
    void send(String message);
}

class EmailNotification implements Notification {
    public void send(String message) {
        System.out.println("Email: " + message);
    }
}

class SMSNotification implements Notification {
    public void send(String message) {
        System.out.println("SMS: " + message);
    }
}

class PushNotification implements Notification {
    public void send(String message) {
        System.out.println("Push: " + message);
    }
}

// Simple Factory
class NotificationFactory {
    public static Notification createNotification(String type) {
        switch (type.toLowerCase()) {
            case "email":
                return new EmailNotification();
            case "sms":
                return new SMSNotification();
            case "push":
                return new PushNotification();
            default:
                throw new IllegalArgumentException("Unknown type: " + type);
        }
    }
}

// Usage
Notification notification = NotificationFactory.createNotification("email");
notification.send("Hello!");
```

**Pros:** Simple, centralizes creation logic
**Cons:** Violates Open/Closed Principle (need to modify factory for new types)

---

## Factory Method Pattern (GoF)

The Factory Method is a **design pattern** that uses inheritance.

### Structure:

```
┌─────────────────────────┐          ┌─────────────────────────┐
│        Creator          │          │        Product          │
├─────────────────────────┤          ├─────────────────────────┤
│                         │          │                         │
│ + factoryMethod()       │─────────>│ + operation()           │
│ + someOperation()       │ creates  │                         │
└───────────△─────────────┘          └───────────△─────────────┘
            │                                    │
            │                                    │
┌───────────────────────────┐       ┌───────────────────────────┐
│    ConcreteCreatorA       │       │    ConcreteProductA       │
├───────────────────────────┤       ├───────────────────────────┤
│ + factoryMethod()         │──────>│ + operation()             │
└───────────────────────────┘       └───────────────────────────┘
```

### Example: Document Application

```java
// Product interface
interface Document {
    void open();
    void save();
    void close();
}

// Concrete products
class PDFDocument implements Document {
    public void open() { System.out.println("Opening PDF..."); }
    public void save() { System.out.println("Saving PDF..."); }
    public void close() { System.out.println("Closing PDF..."); }
}

class WordDocument implements Document {
    public void open() { System.out.println("Opening Word..."); }
    public void save() { System.out.println("Saving Word..."); }
    public void close() { System.out.println("Closing Word..."); }
}

class ExcelDocument implements Document {
    public void open() { System.out.println("Opening Excel..."); }
    public void save() { System.out.println("Saving Excel..."); }
    public void close() { System.out.println("Closing Excel..."); }
}

// Creator (abstract)
abstract class Application {
    // Factory Method - subclasses implement this
    protected abstract Document createDocument();
    
    // Common logic that uses the factory method
    public void newDocument() {
        Document doc = createDocument();  // Factory method call
        doc.open();
        System.out.println("Document ready for editing");
    }
    
    public void openDocument(String path) {
        Document doc = createDocument();
        doc.open();
        System.out.println("Opened: " + path);
    }
}

// Concrete Creators
class PDFApplication extends Application {
    @Override
    protected Document createDocument() {
        return new PDFDocument();
    }
}

class WordApplication extends Application {
    @Override
    protected Document createDocument() {
        return new WordDocument();
    }
}

class ExcelApplication extends Application {
    @Override
    protected Document createDocument() {
        return new ExcelDocument();
    }
}

// Usage
Application app = new PDFApplication();
app.newDocument();
// Output: Opening PDF... Document ready for editing

Application wordApp = new WordApplication();
wordApp.newDocument();
// Output: Opening Word... Document ready for editing
```

---

## Real-World Examples

### Example 1: Logistics System

```java
// Product
interface Transport {
    void deliver(String cargo);
    double getCost(double distance);
}

// Concrete Products
class Truck implements Transport {
    public void deliver(String cargo) {
        System.out.println("Delivering " + cargo + " by road in a truck");
    }
    
    public double getCost(double distance) {
        return distance * 1.5;  // $1.5 per km
    }
}

class Ship implements Transport {
    public void deliver(String cargo) {
        System.out.println("Delivering " + cargo + " by sea in a ship");
    }
    
    public double getCost(double distance) {
        return distance * 0.5;  // $0.5 per km (cheaper for long distances)
    }
}

class Airplane implements Transport {
    public void deliver(String cargo) {
        System.out.println("Delivering " + cargo + " by air in a plane");
    }
    
    public double getCost(double distance) {
        return distance * 5.0;  // $5 per km (fast but expensive)
    }
}

// Creator
abstract class Logistics {
    // Factory Method
    protected abstract Transport createTransport();
    
    public void planDelivery(String cargo, double distance) {
        Transport transport = createTransport();
        double cost = transport.getCost(distance);
        System.out.println("Cost estimate: $" + cost);
        transport.deliver(cargo);
    }
}

// Concrete Creators
class RoadLogistics extends Logistics {
    @Override
    protected Transport createTransport() {
        return new Truck();
    }
}

class SeaLogistics extends Logistics {
    @Override
    protected Transport createTransport() {
        return new Ship();
    }
}

class AirLogistics extends Logistics {
    @Override
    protected Transport createTransport() {
        return new Airplane();
    }
}

// Usage
Logistics logistics;

// For local deliveries
logistics = new RoadLogistics();
logistics.planDelivery("Electronics", 100);

// For international deliveries
logistics = new SeaLogistics();
logistics.planDelivery("Cars", 5000);

// For urgent deliveries
logistics = new AirLogistics();
logistics.planDelivery("Medical supplies", 2000);
```

---

### Example 2: Payment Processing

```java
// Product
interface PaymentProcessor {
    boolean processPayment(double amount);
    void refund(String transactionId);
    String getName();
}

// Concrete Products
class CreditCardProcessor implements PaymentProcessor {
    public boolean processPayment(double amount) {
        System.out.println("Processing $" + amount + " via Credit Card");
        // Connect to bank, validate card, etc.
        return true;
    }
    
    public void refund(String transactionId) {
        System.out.println("Refunding transaction: " + transactionId);
    }
    
    public String getName() { return "Credit Card"; }
}

class PayPalProcessor implements PaymentProcessor {
    public boolean processPayment(double amount) {
        System.out.println("Processing $" + amount + " via PayPal");
        // Connect to PayPal API
        return true;
    }
    
    public void refund(String transactionId) {
        System.out.println("PayPal refund: " + transactionId);
    }
    
    public String getName() { return "PayPal"; }
}

class CryptoProcessor implements PaymentProcessor {
    public boolean processPayment(double amount) {
        System.out.println("Processing $" + amount + " via Cryptocurrency");
        // Connect to blockchain
        return true;
    }
    
    public void refund(String transactionId) {
        System.out.println("Crypto refund initiated: " + transactionId);
    }
    
    public String getName() { return "Cryptocurrency"; }
}

// Creator
abstract class PaymentGateway {
    protected abstract PaymentProcessor createProcessor();
    
    public void processOrder(Order order) {
        PaymentProcessor processor = createProcessor();
        System.out.println("Using " + processor.getName() + " gateway");
        
        if (processor.processPayment(order.getTotal())) {
            order.setStatus("PAID");
            System.out.println("Payment successful!");
        } else {
            order.setStatus("FAILED");
            System.out.println("Payment failed!");
        }
    }
}

// Concrete Creators
class CreditCardGateway extends PaymentGateway {
    @Override
    protected PaymentProcessor createProcessor() {
        return new CreditCardProcessor();
    }
}

class PayPalGateway extends PaymentGateway {
    @Override
    protected PaymentProcessor createProcessor() {
        return new PayPalProcessor();
    }
}

class CryptoGateway extends PaymentGateway {
    @Override
    protected PaymentProcessor createProcessor() {
        return new CryptoProcessor();
    }
}
```

---

### Example 3: Cross-Platform UI

```java
// Product interfaces
interface Button {
    void render();
    void onClick(Runnable action);
}

interface TextField {
    void render();
    String getValue();
}

// Windows products
class WindowsButton implements Button {
    public void render() {
        System.out.println("Rendering Windows style button");
    }
    
    public void onClick(Runnable action) {
        System.out.println("Windows click handler");
        action.run();
    }
}

// Mac products
class MacButton implements Button {
    public void render() {
        System.out.println("Rendering macOS style button");
    }
    
    public void onClick(Runnable action) {
        System.out.println("Mac click handler");
        action.run();
    }
}

// Linux products
class LinuxButton implements Button {
    public void render() {
        System.out.println("Rendering Linux/GTK style button");
    }
    
    public void onClick(Runnable action) {
        System.out.println("Linux click handler");
        action.run();
    }
}

// Creator
abstract class Dialog {
    protected abstract Button createButton();
    
    public void render() {
        Button button = createButton();
        button.render();
        button.onClick(() -> System.out.println("Dialog closed"));
    }
}

// Concrete Creators
class WindowsDialog extends Dialog {
    @Override
    protected Button createButton() {
        return new WindowsButton();
    }
}

class MacDialog extends Dialog {
    @Override
    protected Button createButton() {
        return new MacButton();
    }
}

class LinuxDialog extends Dialog {
    @Override
    protected Button createButton() {
        return new LinuxButton();
    }
}

// Client code
public class Application {
    private Dialog dialog;
    
    public void initialize(String os) {
        switch (os.toLowerCase()) {
            case "windows":
                dialog = new WindowsDialog();
                break;
            case "mac":
                dialog = new MacDialog();
                break;
            default:
                dialog = new LinuxDialog();
        }
    }
    
    public void run() {
        dialog.render();
    }
}
```

---

## Parameterized Factory Method

Factory method can take parameters:

```java
abstract class VehicleFactory {
    // Parameterized factory method
    protected abstract Vehicle createVehicle(String type);
    
    public Vehicle orderVehicle(String type) {
        Vehicle vehicle = createVehicle(type);
        vehicle.assemble();
        vehicle.paint();
        vehicle.test();
        return vehicle;
    }
}

class CarFactory extends VehicleFactory {
    @Override
    protected Vehicle createVehicle(String type) {
        switch (type.toLowerCase()) {
            case "sedan": return new Sedan();
            case "suv": return new SUV();
            case "sports": return new SportsCar();
            default: throw new IllegalArgumentException("Unknown car type");
        }
    }
}

class MotorcycleFactory extends VehicleFactory {
    @Override
    protected Vehicle createVehicle(String type) {
        switch (type.toLowerCase()) {
            case "cruiser": return new Cruiser();
            case "sport": return new SportBike();
            default: throw new IllegalArgumentException("Unknown bike type");
        }
    }
}
```

---

## Factory Method in Java Standard Library

### 1. java.util.Calendar
```java
Calendar calendar = Calendar.getInstance();  // Factory method
// Returns GregorianCalendar, JapaneseImperialCalendar, etc.
```

### 2. java.text.NumberFormat
```java
NumberFormat nf = NumberFormat.getInstance();
NumberFormat currency = NumberFormat.getCurrencyInstance();
```

### 3. java.util.ResourceBundle
```java
ResourceBundle bundle = ResourceBundle.getBundle("messages");
```

### 4. java.nio.charset.Charset
```java
Charset charset = Charset.forName("UTF-8");
```

---

## Factory Method vs Simple Factory

| Aspect | Simple Factory | Factory Method |
|--------|----------------|----------------|
| Structure | Single class | Class hierarchy |
| Extension | Modify factory | Add new creator subclass |
| OCP | Violates | Follows |
| Flexibility | Limited | High |
| Complexity | Low | Medium |

---

## When to Use Factory Method

### ✅ Use When:
1. **Don't know exact types in advance**
   - Types determined at runtime

2. **Want to allow extension**
   - New types without modifying existing code

3. **Object creation has complex logic**
   - Centralize and reuse creation logic

4. **Working with frameworks**
   - Let users extend your framework

### ❌ Don't Use When:
1. Only one type of object needed
2. Simple object creation
3. No variation expected

---

## UML Diagram

```
        ┌────────────────────────────────┐
        │           Creator              │
        ├────────────────────────────────┤
        │ + factoryMethod(): Product     │ {abstract}
        │ + operation(): void            │
        └───────────────△────────────────┘
                        │
           ┌────────────┴────────────┐
           │                         │
┌──────────────────────┐  ┌──────────────────────┐
│  ConcreteCreatorA    │  │  ConcreteCreatorB    │
├──────────────────────┤  ├──────────────────────┤
│ + factoryMethod()    │  │ + factoryMethod()    │
│   : ProductA         │  │   : ProductB         │
└──────────────────────┘  └──────────────────────┘
           │                         │
           │ creates                 │ creates
           ▼                         ▼
┌──────────────────────┐  ┌──────────────────────┐
│      ProductA        │  │      ProductB        │
└──────────────────────┘  └──────────────────────┘
           △                         △
           └───────────┬─────────────┘
                       │
           ┌───────────────────────┐
           │    <<interface>>      │
           │       Product         │
           └───────────────────────┘
```

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Intent** | Let subclasses decide which class to create |
| **Key Idea** | Abstract factory method, concrete subclasses implement |
| **Benefit** | Add new products without modifying existing code |
| **Follows** | Open/Closed Principle, Dependency Inversion |
| **Drawback** | More classes to maintain |

### Remember:
- Factory Method uses **inheritance**
- Abstract creator defines the factory method
- Concrete creators implement it with specific products
- Client code works with creator abstraction

---

**Next: Builder Pattern →**
