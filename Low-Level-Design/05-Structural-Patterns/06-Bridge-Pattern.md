# Bridge Design Pattern

## Intent

> **Decouple an abstraction from its implementation so that the two can vary independently.**

---

## The Problem

You have **two dimensions** that can change:
- A **Shape** can be Circle, Square, Rectangle
- A **Color** can be Red, Blue, Green

### With Inheritance (Explosive!)

```java
// Explosion of classes! 🔥
class RedCircle extends Circle { }
class BlueCircle extends Circle { }
class GreenCircle extends Circle { }
class RedSquare extends Square { }
class BlueSquare extends Square { }
class GreenSquare extends Square { }
// And it gets worse with more colors/shapes!
```

For **M shapes × N colors** = **M × N classes**!

### With Bridge (Clean!)

```java
// Only M + N classes
// Shapes: Circle, Square, Rectangle (3 classes)
// Colors: Red, Blue, Green (3 classes)
// Total: 6 classes instead of 9!
```

---

## Simple Analogy

Think of a **TV and Remote Control**:
- **Abstraction**: Remote control (what user interacts with)
- **Implementation**: TV internals (how it works)
- You can have different remotes (basic, advanced)
- You can have different TVs (Sony, Samsung, LG)
- Any remote can work with any TV!

Or think of **Drivers and Devices**:
- Same printer driver interface
- Different printer hardware implementations
- Bridge connects software to hardware

---

## Structure

```
┌──────────────────────────────────────┐
│            Abstraction               │
├──────────────────────────────────────┤       ┌───────────────────────┐
│ - implementor: Implementor           │──────>│     Implementor       │
├──────────────────────────────────────┤       ├───────────────────────┤
│ + operation()                        │       │ + operationImpl()     │
└────────────────△─────────────────────┘       └───────────△───────────┘
                 │                                         │
                 │                                ┌────────┴────────┐
     ┌───────────────────────┐                   │                  │
     │   RefinedAbstraction  │           ┌──────────────┐  ┌──────────────┐
     ├───────────────────────┤           │ ConcreteImplA │  │ ConcreteImplB │
     │ + operation()         │           └──────────────┘  └──────────────┘
     └───────────────────────┘

Abstraction uses Implementor (composition, not inheritance!)
```

---

## Basic Example: Shapes and Colors

```java
// IMPLEMENTOR - Color implementation
interface Color {
    String fill();
}

// Concrete Implementors
class RedColor implements Color {
    @Override
    public String fill() {
        return "Red";
    }
}

class BlueColor implements Color {
    @Override
    public String fill() {
        return "Blue";
    }
}

class GreenColor implements Color {
    @Override
    public String fill() {
        return "Green";
    }
}

// ABSTRACTION - Shape
abstract class Shape {
    protected Color color;  // Bridge to implementation
    
    public Shape(Color color) {
        this.color = color;
    }
    
    abstract void draw();
}

// Refined Abstractions
class Circle extends Shape {
    private int radius;
    
    public Circle(int radius, Color color) {
        super(color);
        this.radius = radius;
    }
    
    @Override
    void draw() {
        System.out.println("Drawing Circle with radius " + radius + 
                          " in " + color.fill() + " color");
    }
}

class Square extends Shape {
    private int side;
    
    public Square(int side, Color color) {
        super(color);
        this.side = side;
    }
    
    @Override
    void draw() {
        System.out.println("Drawing Square with side " + side + 
                          " in " + color.fill() + " color");
    }
}

class Rectangle extends Shape {
    private int width, height;
    
    public Rectangle(int width, int height, Color color) {
        super(color);
        this.width = width;
        this.height = height;
    }
    
    @Override
    void draw() {
        System.out.println("Drawing Rectangle " + width + "x" + height + 
                          " in " + color.fill() + " color");
    }
}

// Usage
public class Main {
    public static void main(String[] args) {
        // Any shape + any color combination!
        Shape redCircle = new Circle(5, new RedColor());
        Shape blueSquare = new Square(10, new BlueColor());
        Shape greenRectangle = new Rectangle(4, 6, new GreenColor());
        Shape blueCircle = new Circle(8, new BlueColor());
        
        redCircle.draw();       // Circle radius 5 in Red
        blueSquare.draw();      // Square side 10 in Blue
        greenRectangle.draw();  // Rectangle 4x6 in Green
        blueCircle.draw();      // Circle radius 8 in Blue
        
        // Adding new color? Just add ONE class!
        // Adding new shape? Just add ONE class!
    }
}
```

---

## Real-World Examples

### Example 1: Remote Control and Devices

```java
// IMPLEMENTOR - Device interface
interface Device {
    boolean isEnabled();
    void enable();
    void disable();
    int getVolume();
    void setVolume(int volume);
    int getChannel();
    void setChannel(int channel);
}

// Concrete Implementors
class TV implements Device {
    private boolean on = false;
    private int volume = 30;
    private int channel = 1;
    
    @Override
    public boolean isEnabled() { return on; }
    
    @Override
    public void enable() {
        on = true;
        System.out.println("TV is ON");
    }
    
    @Override
    public void disable() {
        on = false;
        System.out.println("TV is OFF");
    }
    
    @Override
    public int getVolume() { return volume; }
    
    @Override
    public void setVolume(int volume) {
        this.volume = Math.max(0, Math.min(100, volume));
        System.out.println("TV volume: " + this.volume);
    }
    
    @Override
    public int getChannel() { return channel; }
    
    @Override
    public void setChannel(int channel) {
        this.channel = channel;
        System.out.println("TV channel: " + channel);
    }
}

class Radio implements Device {
    private boolean on = false;
    private int volume = 20;
    private int channel = 88;  // FM frequency
    
    @Override
    public boolean isEnabled() { return on; }
    
    @Override
    public void enable() {
        on = true;
        System.out.println("Radio is ON");
    }
    
    @Override
    public void disable() {
        on = false;
        System.out.println("Radio is OFF");
    }
    
    @Override
    public int getVolume() { return volume; }
    
    @Override
    public void setVolume(int volume) {
        this.volume = Math.max(0, Math.min(100, volume));
        System.out.println("Radio volume: " + this.volume);
    }
    
    @Override
    public int getChannel() { return channel; }
    
    @Override
    public void setChannel(int channel) {
        this.channel = channel;
        System.out.println("Radio frequency: " + channel + " FM");
    }
}

// ABSTRACTION - Remote control
abstract class RemoteControl {
    protected Device device;  // Bridge!
    
    public RemoteControl(Device device) {
        this.device = device;
    }
    
    public void togglePower() {
        if (device.isEnabled()) {
            device.disable();
        } else {
            device.enable();
        }
    }
    
    public void volumeUp() {
        device.setVolume(device.getVolume() + 10);
    }
    
    public void volumeDown() {
        device.setVolume(device.getVolume() - 10);
    }
    
    public void channelUp() {
        device.setChannel(device.getChannel() + 1);
    }
    
    public void channelDown() {
        device.setChannel(device.getChannel() - 1);
    }
}

// Refined Abstraction - Basic remote
class BasicRemote extends RemoteControl {
    public BasicRemote(Device device) {
        super(device);
    }
    // Uses inherited methods only
}

// Refined Abstraction - Advanced remote with extra features
class AdvancedRemote extends RemoteControl {
    public AdvancedRemote(Device device) {
        super(device);
    }
    
    public void mute() {
        System.out.println("Muting...");
        device.setVolume(0);
    }
    
    public void setChannel(int channel) {
        device.setChannel(channel);
    }
}

// Usage
public class RemoteDemo {
    public static void main(String[] args) {
        System.out.println("=== Basic Remote with TV ===");
        Device tv = new TV();
        RemoteControl basicTvRemote = new BasicRemote(tv);
        
        basicTvRemote.togglePower();  // Turn on
        basicTvRemote.volumeUp();
        basicTvRemote.channelUp();
        basicTvRemote.togglePower();  // Turn off
        
        System.out.println("\n=== Advanced Remote with Radio ===");
        Device radio = new Radio();
        AdvancedRemote advancedRadioRemote = new AdvancedRemote(radio);
        
        advancedRadioRemote.togglePower();
        advancedRadioRemote.setChannel(102);
        advancedRadioRemote.volumeUp();
        advancedRadioRemote.mute();  // Advanced feature!
        
        System.out.println("\n=== Same Advanced Remote with TV ===");
        AdvancedRemote advancedTvRemote = new AdvancedRemote(new TV());
        advancedTvRemote.togglePower();
        advancedTvRemote.mute();
    }
}
```

---

### Example 2: Notification System (Cross-Platform)

```java
// IMPLEMENTOR - Message sending mechanism
interface MessageSender {
    void sendMessage(String message, String recipient);
}

// Concrete Implementors - Different platforms
class EmailSender implements MessageSender {
    @Override
    public void sendMessage(String message, String recipient) {
        System.out.println("📧 Email to " + recipient + ": " + message);
    }
}

class SMSSender implements MessageSender {
    @Override
    public void sendMessage(String message, String recipient) {
        System.out.println("📱 SMS to " + recipient + ": " + message);
    }
}

class PushNotificationSender implements MessageSender {
    @Override
    public void sendMessage(String message, String recipient) {
        System.out.println("🔔 Push to " + recipient + ": " + message);
    }
}

class SlackSender implements MessageSender {
    @Override
    public void sendMessage(String message, String recipient) {
        System.out.println("💬 Slack to #" + recipient + ": " + message);
    }
}

// ABSTRACTION - Notification types
abstract class Notification {
    protected MessageSender sender;  // Bridge!
    
    public Notification(MessageSender sender) {
        this.sender = sender;
    }
    
    abstract void send(String recipient, String title, String body);
}

// Refined Abstractions - Different notification types
class AlertNotification extends Notification {
    public AlertNotification(MessageSender sender) {
        super(sender);
    }
    
    @Override
    void send(String recipient, String title, String body) {
        String message = "🚨 ALERT: " + title + "\n" + body;
        sender.sendMessage(message, recipient);
    }
}

class PromotionalNotification extends Notification {
    public PromotionalNotification(MessageSender sender) {
        super(sender);
    }
    
    @Override
    void send(String recipient, String title, String body) {
        String message = "🎉 " + title + "\n" + body + "\n[Reply STOP to unsubscribe]";
        sender.sendMessage(message, recipient);
    }
}

class TransactionalNotification extends Notification {
    public TransactionalNotification(MessageSender sender) {
        super(sender);
    }
    
    @Override
    void send(String recipient, String title, String body) {
        String message = "📝 " + title + "\n" + body + "\nTransaction ID: TXN" + 
                        System.currentTimeMillis();
        sender.sendMessage(message, recipient);
    }
}

// Usage
public class NotificationSystem {
    public static void main(String[] args) {
        // Alert via Email
        Notification alertEmail = new AlertNotification(new EmailSender());
        alertEmail.send("admin@company.com", "Server Down", "Production server is unresponsive");
        
        // Alert via SMS
        Notification alertSMS = new AlertNotification(new SMSSender());
        alertSMS.send("+1234567890", "Server Down", "Production server is unresponsive");
        
        // Promotional via Push
        Notification promoPush = new PromotionalNotification(new PushNotificationSender());
        promoPush.send("user123", "Flash Sale!", "50% off on all items");
        
        // Transaction via Email
        Notification transEmail = new TransactionalNotification(new EmailSender());
        transEmail.send("customer@email.com", "Payment Received", "Your payment of $99.99 was successful");
        
        // Transaction via Slack
        Notification transSlack = new TransactionalNotification(new SlackSender());
        transSlack.send("payments", "New Payment", "Customer paid $99.99");
    }
}
```

---

### Example 3: Database and Data Format

```java
// IMPLEMENTOR - Data format
interface DataFormat {
    String format(Map<String, Object> data);
    Map<String, Object> parse(String data);
}

class JSONFormat implements DataFormat {
    @Override
    public String format(Map<String, Object> data) {
        StringBuilder json = new StringBuilder("{");
        boolean first = true;
        for (Map.Entry<String, Object> entry : data.entrySet()) {
            if (!first) json.append(",");
            json.append("\"").append(entry.getKey()).append("\":\"")
                .append(entry.getValue()).append("\"");
            first = false;
        }
        json.append("}");
        return json.toString();
    }
    
    @Override
    public Map<String, Object> parse(String data) {
        // Simplified parsing
        Map<String, Object> result = new HashMap<>();
        // ... parsing logic
        return result;
    }
}

class XMLFormat implements DataFormat {
    @Override
    public String format(Map<String, Object> data) {
        StringBuilder xml = new StringBuilder("<data>");
        for (Map.Entry<String, Object> entry : data.entrySet()) {
            xml.append("<").append(entry.getKey()).append(">")
               .append(entry.getValue())
               .append("</").append(entry.getKey()).append(">");
        }
        xml.append("</data>");
        return xml.toString();
    }
    
    @Override
    public Map<String, Object> parse(String data) {
        Map<String, Object> result = new HashMap<>();
        // ... parsing logic
        return result;
    }
}

class CSVFormat implements DataFormat {
    @Override
    public String format(Map<String, Object> data) {
        StringBuilder csv = new StringBuilder();
        csv.append(String.join(",", data.keySet())).append("\n");
        csv.append(String.join(",", data.values().stream()
            .map(Object::toString).toArray(String[]::new)));
        return csv.toString();
    }
    
    @Override
    public Map<String, Object> parse(String data) {
        Map<String, Object> result = new HashMap<>();
        // ... parsing logic
        return result;
    }
}

// ABSTRACTION - Database export
abstract class DataExport {
    protected DataFormat format;  // Bridge!
    
    public DataExport(DataFormat format) {
        this.format = format;
    }
    
    abstract String export();
}

// Refined Abstractions
class UserExport extends DataExport {
    private List<Map<String, Object>> users;
    
    public UserExport(DataFormat format, List<Map<String, Object>> users) {
        super(format);
        this.users = users;
    }
    
    @Override
    String export() {
        StringBuilder result = new StringBuilder();
        result.append("=== User Export ===\n");
        for (Map<String, Object> user : users) {
            result.append(format.format(user)).append("\n");
        }
        return result.toString();
    }
}

class OrderExport extends DataExport {
    private List<Map<String, Object>> orders;
    
    public OrderExport(DataFormat format, List<Map<String, Object>> orders) {
        super(format);
        this.orders = orders;
    }
    
    @Override
    String export() {
        StringBuilder result = new StringBuilder();
        result.append("=== Order Export ===\n");
        for (Map<String, Object> order : orders) {
            result.append(format.format(order)).append("\n");
        }
        return result.toString();
    }
}

// Usage
public class ExportSystem {
    public static void main(String[] args) {
        List<Map<String, Object>> users = new ArrayList<>();
        users.add(Map.of("name", "John", "email", "john@email.com"));
        users.add(Map.of("name", "Jane", "email", "jane@email.com"));
        
        // Export users as JSON
        DataExport jsonExport = new UserExport(new JSONFormat(), users);
        System.out.println(jsonExport.export());
        
        // Export users as XML
        DataExport xmlExport = new UserExport(new XMLFormat(), users);
        System.out.println(xmlExport.export());
        
        // Export users as CSV
        DataExport csvExport = new UserExport(new CSVFormat(), users);
        System.out.println(csvExport.export());
    }
}
```

---

## When to Use Bridge Pattern

### ✅ Use When:
1. **Two independent dimensions** that can vary
2. **Avoid class explosion** from combinations
3. **Switch implementations** at runtime
4. **Hide implementation details** from client

### ❌ Don't Use When:
1. Only one dimension of variation
2. Dimensions are tightly coupled
3. Simple cases where inheritance works

---

## Bridge vs Other Patterns

| Pattern | Purpose |
|---------|---------|
| **Bridge** | Separate abstraction from implementation |
| **Adapter** | Make incompatible interfaces work together |
| **Strategy** | Change algorithm at runtime |
| **State** | Change behavior based on state |

### Key Difference:
- **Bridge** is designed upfront to separate dimensions
- **Adapter** is applied later to fix compatibility

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Intent** | Separate abstraction from implementation |
| **Problem** | M × N class explosion from combinations |
| **Solution** | Composition instead of inheritance |
| **Result** | M + N classes, flexible combinations |

### Remember:
- Abstraction contains reference to Implementor (Bridge)
- Both hierarchies can grow independently
- Changes in one don't affect the other
- **"Prefer composition over inheritance"** in action!

---

**Next: Flyweight Pattern →**
