# Decorator Design Pattern

## Intent

> **Attach additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing for extending functionality.**

---

## The Problem

You want to add behavior to objects:
- At **runtime** (not compile time)
- **Without modifying** the original class
- Using **composition** instead of inheritance
- Combine **multiple behaviors** flexibly

### Why Not Just Use Inheritance?

```java
// Explosion of subclasses!
class Coffee { }
class CoffeeWithMilk extends Coffee { }
class CoffeeWithSugar extends Coffee { }
class CoffeeWithMilkAndSugar extends Coffee { }
class CoffeeWithMilkAndSugarAndWhip extends Coffee { }
class CoffeeWithCaramel extends Coffee { }
class CoffeeWithMilkAndCaramel extends Coffee { }
// ... endless combinations! 💥
```

---

## Simple Analogy

Think of **Pizza Toppings**:
- Start with basic pizza ($10)
- Add cheese (+$2)
- Add pepperoni (+$3)
- Add mushrooms (+$1.5)
- Each topping "decorates" the pizza
- You can combine any toppings in any order

Or think of **Clothing Layers**:
- Base: T-shirt
- Decorator 1: Add sweater
- Decorator 2: Add jacket
- Each layer wraps the previous one

---

## Structure

```
┌──────────────────────────┐
│       Component          │◄──────────────────────┐
├──────────────────────────┤                       │
│ + operation()            │                       │
└───────────△──────────────┘                       │
            │                                      │
   ┌────────┴────────┐                            │
   │                 │                            │
┌──────────────┐  ┌──────────────────────────┐   │
│ Concrete     │  │      Decorator           │   │ wraps
│ Component    │  ├──────────────────────────┤   │
├──────────────┤  │ - component: Component   │───┘
│ + operation()│  ├──────────────────────────┤
└──────────────┘  │ + operation()            │
                  └───────────△──────────────┘
                              │
                    ┌─────────┴─────────┐
                    │                   │
           ┌─────────────────┐  ┌─────────────────┐
           │ ConcreteDecoA   │  │ ConcreteDecoB   │
           ├─────────────────┤  ├─────────────────┤
           │ + operation()   │  │ + operation()   │
           │ + addedBehavior │  │ + addedBehavior │
           └─────────────────┘  └─────────────────┘
```

---

## Basic Example: Coffee Shop

```java
// Component interface
interface Coffee {
    double getCost();
    String getDescription();
}

// Concrete component
class SimpleCoffee implements Coffee {
    @Override
    public double getCost() {
        return 2.00;
    }
    
    @Override
    public String getDescription() {
        return "Simple Coffee";
    }
}

// Base Decorator
abstract class CoffeeDecorator implements Coffee {
    protected Coffee decoratedCoffee;
    
    public CoffeeDecorator(Coffee coffee) {
        this.decoratedCoffee = coffee;
    }
    
    @Override
    public double getCost() {
        return decoratedCoffee.getCost();
    }
    
    @Override
    public String getDescription() {
        return decoratedCoffee.getDescription();
    }
}

// Concrete Decorators
class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public double getCost() {
        return super.getCost() + 0.50;
    }
    
    @Override
    public String getDescription() {
        return super.getDescription() + ", Milk";
    }
}

class SugarDecorator extends CoffeeDecorator {
    public SugarDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public double getCost() {
        return super.getCost() + 0.25;
    }
    
    @Override
    public String getDescription() {
        return super.getDescription() + ", Sugar";
    }
}

class WhipDecorator extends CoffeeDecorator {
    public WhipDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public double getCost() {
        return super.getCost() + 0.75;
    }
    
    @Override
    public String getDescription() {
        return super.getDescription() + ", Whip";
    }
}

class CaramelDecorator extends CoffeeDecorator {
    public CaramelDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public double getCost() {
        return super.getCost() + 0.60;
    }
    
    @Override
    public String getDescription() {
        return super.getDescription() + ", Caramel";
    }
}

// Usage
public class CoffeeShop {
    public static void main(String[] args) {
        // Simple coffee
        Coffee coffee = new SimpleCoffee();
        System.out.println(coffee.getDescription() + " $" + coffee.getCost());
        // Output: Simple Coffee $2.0
        
        // Coffee with milk
        coffee = new MilkDecorator(new SimpleCoffee());
        System.out.println(coffee.getDescription() + " $" + coffee.getCost());
        // Output: Simple Coffee, Milk $2.5
        
        // Fancy coffee with everything!
        coffee = new WhipDecorator(
                    new CaramelDecorator(
                        new MilkDecorator(
                            new SugarDecorator(
                                new SimpleCoffee()))));
        System.out.println(coffee.getDescription() + " $" + coffee.getCost());
        // Output: Simple Coffee, Sugar, Milk, Caramel, Whip $4.1
        
        // Double milk!
        coffee = new MilkDecorator(new MilkDecorator(new SimpleCoffee()));
        System.out.println(coffee.getDescription() + " $" + coffee.getCost());
        // Output: Simple Coffee, Milk, Milk $3.0
    }
}
```

---

## Real-World Examples

### Example 1: Input/Output Streams

```java
// Component
interface DataSource {
    void writeData(String data);
    String readData();
}

// Concrete component
class FileDataSource implements DataSource {
    private String filename;
    
    public FileDataSource(String filename) {
        this.filename = filename;
    }
    
    @Override
    public void writeData(String data) {
        System.out.println("Writing to file " + filename + ": " + data);
        // Actual file writing logic
    }
    
    @Override
    public String readData() {
        System.out.println("Reading from file " + filename);
        return "file content";
    }
}

// Base decorator
class DataSourceDecorator implements DataSource {
    protected DataSource wrappee;
    
    public DataSourceDecorator(DataSource source) {
        this.wrappee = source;
    }
    
    @Override
    public void writeData(String data) {
        wrappee.writeData(data);
    }
    
    @Override
    public String readData() {
        return wrappee.readData();
    }
}

// Encryption decorator
class EncryptionDecorator extends DataSourceDecorator {
    public EncryptionDecorator(DataSource source) {
        super(source);
    }
    
    @Override
    public void writeData(String data) {
        String encrypted = encrypt(data);
        System.out.println("Encrypting data...");
        super.writeData(encrypted);
    }
    
    @Override
    public String readData() {
        String data = super.readData();
        System.out.println("Decrypting data...");
        return decrypt(data);
    }
    
    private String encrypt(String data) {
        // Simple encryption simulation
        return Base64.getEncoder().encodeToString(data.getBytes());
    }
    
    private String decrypt(String data) {
        return new String(Base64.getDecoder().decode(data));
    }
}

// Compression decorator
class CompressionDecorator extends DataSourceDecorator {
    public CompressionDecorator(DataSource source) {
        super(source);
    }
    
    @Override
    public void writeData(String data) {
        String compressed = compress(data);
        System.out.println("Compressing data...");
        super.writeData(compressed);
    }
    
    @Override
    public String readData() {
        String data = super.readData();
        System.out.println("Decompressing data...");
        return decompress(data);
    }
    
    private String compress(String data) {
        // Simplified compression
        return "compressed:" + data.length() + ":" + data;
    }
    
    private String decompress(String data) {
        return data.split(":")[2];
    }
}

// Usage
DataSource source = new FileDataSource("data.txt");
source.writeData("Hello World");
// Output: Writing to file data.txt: Hello World

// With encryption
source = new EncryptionDecorator(new FileDataSource("secure.txt"));
source.writeData("Secret Data");
// Output: Encrypting data... Writing to file secure.txt: U2VjcmV0IERhdGE=

// With encryption AND compression
source = new CompressionDecorator(
            new EncryptionDecorator(
                new FileDataSource("archive.txt")));
source.writeData("Important Data");
// Output: Compressing data... Encrypting data... Writing to file archive.txt
```

---

### Example 2: Text Formatting

```java
// Component
interface Text {
    String getContent();
}

// Concrete component
class PlainText implements Text {
    private String text;
    
    public PlainText(String text) {
        this.text = text;
    }
    
    @Override
    public String getContent() {
        return text;
    }
}

// Base decorator
abstract class TextDecorator implements Text {
    protected Text wrappedText;
    
    public TextDecorator(Text text) {
        this.wrappedText = text;
    }
    
    @Override
    public String getContent() {
        return wrappedText.getContent();
    }
}

// Concrete decorators
class BoldDecorator extends TextDecorator {
    public BoldDecorator(Text text) {
        super(text);
    }
    
    @Override
    public String getContent() {
        return "<b>" + super.getContent() + "</b>";
    }
}

class ItalicDecorator extends TextDecorator {
    public ItalicDecorator(Text text) {
        super(text);
    }
    
    @Override
    public String getContent() {
        return "<i>" + super.getContent() + "</i>";
    }
}

class UnderlineDecorator extends TextDecorator {
    public UnderlineDecorator(Text text) {
        super(text);
    }
    
    @Override
    public String getContent() {
        return "<u>" + super.getContent() + "</u>";
    }
}

class ColorDecorator extends TextDecorator {
    private String color;
    
    public ColorDecorator(Text text, String color) {
        super(text);
        this.color = color;
    }
    
    @Override
    public String getContent() {
        return "<span style='color:" + color + "'>" + super.getContent() + "</span>";
    }
}

// Usage
Text text = new PlainText("Hello World");
System.out.println(text.getContent());
// Output: Hello World

text = new BoldDecorator(new PlainText("Hello World"));
System.out.println(text.getContent());
// Output: <b>Hello World</b>

text = new ItalicDecorator(
        new BoldDecorator(
            new UnderlineDecorator(
                new PlainText("Hello World"))));
System.out.println(text.getContent());
// Output: <i><b><u>Hello World</u></b></i>

text = new ColorDecorator(
        new BoldDecorator(
            new PlainText("Warning")), "red");
System.out.println(text.getContent());
// Output: <span style='color:red'><b>Warning</b></span>
```

---

### Example 3: Notification System

```java
// Component
interface Notifier {
    void send(String message);
}

// Concrete component
class BasicNotifier implements Notifier {
    @Override
    public void send(String message) {
        System.out.println("Basic notification: " + message);
    }
}

// Base decorator
abstract class NotifierDecorator implements Notifier {
    protected Notifier wrappedNotifier;
    
    public NotifierDecorator(Notifier notifier) {
        this.wrappedNotifier = notifier;
    }
    
    @Override
    public void send(String message) {
        wrappedNotifier.send(message);
    }
}

// Add SMS notification
class SMSNotifier extends NotifierDecorator {
    private String phoneNumber;
    
    public SMSNotifier(Notifier notifier, String phoneNumber) {
        super(notifier);
        this.phoneNumber = phoneNumber;
    }
    
    @Override
    public void send(String message) {
        super.send(message);
        sendSMS(message);
    }
    
    private void sendSMS(String message) {
        System.out.println("SMS to " + phoneNumber + ": " + message);
    }
}

// Add Email notification
class EmailNotifier extends NotifierDecorator {
    private String email;
    
    public EmailNotifier(Notifier notifier, String email) {
        super(notifier);
        this.email = email;
    }
    
    @Override
    public void send(String message) {
        super.send(message);
        sendEmail(message);
    }
    
    private void sendEmail(String message) {
        System.out.println("Email to " + email + ": " + message);
    }
}

// Add Slack notification
class SlackNotifier extends NotifierDecorator {
    private String channel;
    
    public SlackNotifier(Notifier notifier, String channel) {
        super(notifier);
        this.channel = channel;
    }
    
    @Override
    public void send(String message) {
        super.send(message);
        sendSlack(message);
    }
    
    private void sendSlack(String message) {
        System.out.println("Slack #" + channel + ": " + message);
    }
}

// Usage
Notifier notifier = new BasicNotifier();
notifier.send("Hello");
// Output: Basic notification: Hello

// SMS + Email
notifier = new EmailNotifier(
            new SMSNotifier(
                new BasicNotifier(), "555-1234"), "user@email.com");
notifier.send("Server is down!");
// Output:
// Basic notification: Server is down!
// SMS to 555-1234: Server is down!
// Email to user@email.com: Server is down!

// All channels!
notifier = new SlackNotifier(
            new EmailNotifier(
                new SMSNotifier(
                    new BasicNotifier(), "555-1234"), 
                "admin@company.com"),
            "alerts");
notifier.send("Critical error!");
// Sends to all: basic, SMS, email, and Slack
```

---

## Java I/O - Perfect Decorator Example

```java
// Java I/O uses Decorator pattern extensively!

// Base component
InputStream inputStream = new FileInputStream("file.txt");

// Add buffering decorator
inputStream = new BufferedInputStream(inputStream);

// Add data reading decorator
DataInputStream dataStream = new DataInputStream(inputStream);

// Now you can read primitives from a buffered file!
int value = dataStream.readInt();

// Chain of decorators:
// FileInputStream → BufferedInputStream → DataInputStream

// Similarly for output:
OutputStream outputStream = new FileOutputStream("output.txt");
outputStream = new BufferedOutputStream(outputStream);
outputStream = new GZIPOutputStream(outputStream);  // Compression!
// Writes to compressed, buffered file
```

---

## When to Use Decorator Pattern

### ✅ Use When:
1. **Add behavior dynamically** at runtime
2. **Combine behaviors** flexibly (any combination)
3. **Avoid subclass explosion** (too many subclasses)
4. **Single Responsibility** - each decorator does one thing

### ❌ Don't Use When:
1. Behavior is fixed at compile time
2. Only one or two variations needed
3. Order of decoration matters in complex ways

---

## Decorator vs Inheritance

| Aspect | Decorator | Inheritance |
|--------|-----------|-------------|
| **When** | Runtime | Compile time |
| **Flexibility** | High (combine any decorators) | Low (fixed hierarchy) |
| **Classes needed** | N decorators for N behaviors | 2^N subclasses for combinations |
| **Modification** | Non-invasive | Requires changing class |

---

## Decorator vs Other Patterns

| Pattern | Purpose |
|---------|---------|
| **Decorator** | Add behavior dynamically |
| **Adapter** | Convert interface |
| **Proxy** | Control access |
| **Composite** | Tree structure of objects |

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Intent** | Add responsibilities dynamically |
| **Key Idea** | Wrap objects in decorator layers |
| **Benefits** | Flexible, combinable, single responsibility |
| **Structure** | Component → Decorator → ConcreteDecorators |

### Remember:
- Decorators wrap other components
- Each decorator adds ONE behavior
- Decorators are stackable (chainable)
- Both component and decorator share interface

---

**Next: Facade Pattern →**
