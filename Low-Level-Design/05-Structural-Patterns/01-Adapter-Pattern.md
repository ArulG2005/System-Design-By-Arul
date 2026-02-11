# Adapter Design Pattern

## Intent

> **Convert the interface of a class into another interface clients expect. Adapter lets classes work together that couldn't otherwise because of incompatible interfaces.**

---

## The Problem

You have:
- **Existing code** that works with one interface
- **New code/library** that has a different interface
- You want them to work together **without modifying either**

---

## Simple Analogy

Think of a **Power Adapter for Travel**:
- Your laptop has a US plug
- The wall socket is European
- The adapter converts US plug → European socket
- Neither the laptop nor the wall socket is modified!

Or think of **Language Translator**:
- Speaker speaks English
- Listener understands French
- Translator (adapter) converts English → French

---

## Types of Adapters

| Type | Description |
|------|-------------|
| **Object Adapter** | Uses composition (wraps the adaptee) |
| **Class Adapter** | Uses inheritance (extends the adaptee) |

In Java, **Object Adapter is preferred** (composition over inheritance).

---

## Structure

### Object Adapter (Composition)

```
┌───────────────────┐
│      Client       │
└─────────┬─────────┘
          │ uses
          ▼
┌───────────────────┐        ┌───────────────────┐
│  Target Interface │        │     Adaptee       │
├───────────────────┤        ├───────────────────┤
│ + request()       │        │ + specificReq()   │
└─────────△─────────┘        └───────────────────┘
          │                          △
          │ implements               │ contains
┌─────────────────────────────────────────────────┐
│                   Adapter                        │
├─────────────────────────────────────────────────┤
│ - adaptee: Adaptee                              │
├─────────────────────────────────────────────────┤
│ + request()  →  adaptee.specificReq()           │
└─────────────────────────────────────────────────┘
```

---

## Basic Example

### The Problem

```java
// Our application uses this interface
interface MediaPlayer {
    void play(String filename);
}

// We have a new advanced player with different interface
class VLCPlayer {
    public void playVLC(String filename) {
        System.out.println("Playing VLC video: " + filename);
    }
}

class MP4Player {
    public void playMP4(String filename) {
        System.out.println("Playing MP4 video: " + filename);
    }
}

// Problem: VLCPlayer doesn't implement MediaPlayer!
// We can't use it directly with existing code.
```

### The Solution: Adapter

```java
// Target Interface
interface MediaPlayer {
    void play(String filename);
}

// Adaptees (third-party or legacy code)
class VLCPlayer {
    public void playVLC(String filename) {
        System.out.println("Playing VLC video: " + filename);
    }
}

class MP4Player {
    public void playMP4(String filename) {
        System.out.println("Playing MP4 video: " + filename);
    }
}

// Adapter for VLC
class VLCAdapter implements MediaPlayer {
    private VLCPlayer vlcPlayer;
    
    public VLCAdapter(VLCPlayer vlcPlayer) {
        this.vlcPlayer = vlcPlayer;
    }
    
    @Override
    public void play(String filename) {
        vlcPlayer.playVLC(filename);  // Delegate to adaptee
    }
}

// Adapter for MP4
class MP4Adapter implements MediaPlayer {
    private MP4Player mp4Player;
    
    public MP4Adapter(MP4Player mp4Player) {
        this.mp4Player = mp4Player;
    }
    
    @Override
    public void play(String filename) {
        mp4Player.playMP4(filename);  // Delegate to adaptee
    }
}

// Client code works with MediaPlayer interface
class AudioPlayer {
    public void play(String audioType, String filename) {
        MediaPlayer player;
        
        if (audioType.equalsIgnoreCase("vlc")) {
            player = new VLCAdapter(new VLCPlayer());
        } else if (audioType.equalsIgnoreCase("mp4")) {
            player = new MP4Adapter(new MP4Player());
        } else {
            throw new IllegalArgumentException("Unknown format: " + audioType);
        }
        
        player.play(filename);
    }
}

// Usage
AudioPlayer audioPlayer = new AudioPlayer();
audioPlayer.play("vlc", "movie.vlc");
audioPlayer.play("mp4", "video.mp4");
```

---

## Real-World Examples

### Example 1: Payment Gateway Adapter

```java
// Your application's payment interface
interface PaymentProcessor {
    boolean processPayment(double amount);
    String getPaymentStatus(String transactionId);
    void refund(String transactionId);
}

// Third-party payment services (different interfaces)
class PayPalAPI {
    public boolean makePayment(double dollars) {
        System.out.println("PayPal: Processing $" + dollars);
        return true;
    }
    
    public String checkStatus(String id) {
        return "PayPal status: COMPLETED for " + id;
    }
    
    public void issueRefund(String id) {
        System.out.println("PayPal: Refunding " + id);
    }
}

class StripeAPI {
    public int chargeCard(int cents) {  // Different: uses cents!
        System.out.println("Stripe: Charging " + cents + " cents");
        return 1;  // Returns status code
    }
    
    public String getChargeStatus(String chargeId) {
        return "Stripe charge " + chargeId + ": succeeded";
    }
    
    public void createRefund(String chargeId, int amount) {
        System.out.println("Stripe: Refunding " + amount + " cents");
    }
}

class RazorpayAPI {
    public boolean capturePayment(double amountINR, String currency) {
        System.out.println("Razorpay: Capturing " + amountINR + " " + currency);
        return true;
    }
    
    public String fetchPaymentStatus(String paymentId) {
        return "Razorpay payment " + paymentId + ": captured";
    }
    
    public void refundPayment(String paymentId) {
        System.out.println("Razorpay: Initiating refund for " + paymentId);
    }
}

// Adapters
class PayPalAdapter implements PaymentProcessor {
    private PayPalAPI payPal;
    
    public PayPalAdapter(PayPalAPI payPal) {
        this.payPal = payPal;
    }
    
    @Override
    public boolean processPayment(double amount) {
        return payPal.makePayment(amount);
    }
    
    @Override
    public String getPaymentStatus(String transactionId) {
        return payPal.checkStatus(transactionId);
    }
    
    @Override
    public void refund(String transactionId) {
        payPal.issueRefund(transactionId);
    }
}

class StripeAdapter implements PaymentProcessor {
    private StripeAPI stripe;
    
    public StripeAdapter(StripeAPI stripe) {
        this.stripe = stripe;
    }
    
    @Override
    public boolean processPayment(double amount) {
        int cents = (int) (amount * 100);  // Convert dollars to cents
        int status = stripe.chargeCard(cents);
        return status == 1;
    }
    
    @Override
    public String getPaymentStatus(String transactionId) {
        return stripe.getChargeStatus(transactionId);
    }
    
    @Override
    public void refund(String transactionId) {
        stripe.createRefund(transactionId, 0);  // Full refund
    }
}

class RazorpayAdapter implements PaymentProcessor {
    private RazorpayAPI razorpay;
    private String currency;
    
    public RazorpayAdapter(RazorpayAPI razorpay, String currency) {
        this.razorpay = razorpay;
        this.currency = currency;
    }
    
    @Override
    public boolean processPayment(double amount) {
        return razorpay.capturePayment(amount, currency);
    }
    
    @Override
    public String getPaymentStatus(String transactionId) {
        return razorpay.fetchPaymentStatus(transactionId);
    }
    
    @Override
    public void refund(String transactionId) {
        razorpay.refundPayment(transactionId);
    }
}

// Client code - works with any payment processor!
class CheckoutService {
    private PaymentProcessor paymentProcessor;
    
    public CheckoutService(PaymentProcessor paymentProcessor) {
        this.paymentProcessor = paymentProcessor;
    }
    
    public void checkout(double amount) {
        if (paymentProcessor.processPayment(amount)) {
            System.out.println("Checkout successful!");
        } else {
            System.out.println("Payment failed!");
        }
    }
}

// Usage - easily switch payment providers
PaymentProcessor paypal = new PayPalAdapter(new PayPalAPI());
PaymentProcessor stripe = new StripeAdapter(new StripeAPI());
PaymentProcessor razorpay = new RazorpayAdapter(new RazorpayAPI(), "INR");

CheckoutService checkout = new CheckoutService(paypal);
checkout.checkout(99.99);

// Switch to Stripe - no changes to CheckoutService!
checkout = new CheckoutService(stripe);
checkout.checkout(49.99);
```

---

### Example 2: Data Format Adapter

```java
// Your application works with JSON
interface DataReader {
    List<Map<String, Object>> readData(String source);
}

// But you have data in different formats
class XMLParser {
    public Document parseXML(String xmlContent) {
        System.out.println("Parsing XML...");
        // Parse XML and return Document
        return null;  // Simplified
    }
    
    public List<Element> getElements(Document doc, String tagName) {
        // Get elements from XML
        return new ArrayList<>();
    }
}

class CSVParser {
    public String[][] parseCSV(String csvContent, String delimiter) {
        System.out.println("Parsing CSV with delimiter: " + delimiter);
        // Parse CSV and return 2D array
        return new String[0][0];
    }
}

class YAMLParser {
    public Object loadYAML(String yamlContent) {
        System.out.println("Loading YAML...");
        // Parse YAML
        return null;
    }
}

// Adapters
class XMLDataAdapter implements DataReader {
    private XMLParser xmlParser;
    
    public XMLDataAdapter(XMLParser xmlParser) {
        this.xmlParser = xmlParser;
    }
    
    @Override
    public List<Map<String, Object>> readData(String source) {
        Document doc = xmlParser.parseXML(source);
        // Convert XML to List<Map>
        List<Map<String, Object>> result = new ArrayList<>();
        // Conversion logic here...
        return result;
    }
}

class CSVDataAdapter implements DataReader {
    private CSVParser csvParser;
    private String delimiter;
    
    public CSVDataAdapter(CSVParser csvParser, String delimiter) {
        this.csvParser = csvParser;
        this.delimiter = delimiter;
    }
    
    @Override
    public List<Map<String, Object>> readData(String source) {
        String[][] data = csvParser.parseCSV(source, delimiter);
        // Convert CSV to List<Map>
        List<Map<String, Object>> result = new ArrayList<>();
        // Assume first row is headers
        // Conversion logic here...
        return result;
    }
}

// Client code
class DataProcessor {
    private DataReader reader;
    
    public DataProcessor(DataReader reader) {
        this.reader = reader;
    }
    
    public void process(String data) {
        List<Map<String, Object>> records = reader.readData(data);
        for (Map<String, Object> record : records) {
            System.out.println("Processing: " + record);
        }
    }
}
```

---

### Example 3: Legacy System Adapter

```java
// New system interface
interface ModernLogger {
    void debug(String message);
    void info(String message);
    void warn(String message);
    void error(String message, Throwable t);
}

// Legacy logging system with different interface
class LegacyLogger {
    public void writeLog(int level, String msg) {
        String levelStr;
        switch (level) {
            case 0: levelStr = "DEBUG"; break;
            case 1: levelStr = "INFO"; break;
            case 2: levelStr = "WARN"; break;
            case 3: levelStr = "ERROR"; break;
            default: levelStr = "UNKNOWN";
        }
        System.out.println("[LEGACY] " + levelStr + ": " + msg);
    }
    
    public void writeException(String msg, String stackTrace) {
        System.out.println("[LEGACY] EXCEPTION: " + msg + "\n" + stackTrace);
    }
}

// Adapter
class LegacyLoggerAdapter implements ModernLogger {
    private LegacyLogger legacyLogger;
    
    public LegacyLoggerAdapter(LegacyLogger legacyLogger) {
        this.legacyLogger = legacyLogger;
    }
    
    @Override
    public void debug(String message) {
        legacyLogger.writeLog(0, message);
    }
    
    @Override
    public void info(String message) {
        legacyLogger.writeLog(1, message);
    }
    
    @Override
    public void warn(String message) {
        legacyLogger.writeLog(2, message);
    }
    
    @Override
    public void error(String message, Throwable t) {
        legacyLogger.writeLog(3, message);
        if (t != null) {
            StringWriter sw = new StringWriter();
            t.printStackTrace(new PrintWriter(sw));
            legacyLogger.writeException(message, sw.toString());
        }
    }
}

// Modern code works with legacy system seamlessly
class ModernService {
    private ModernLogger logger;
    
    public ModernService(ModernLogger logger) {
        this.logger = logger;
    }
    
    public void doWork() {
        logger.info("Starting work...");
        try {
            // Work logic
            logger.debug("Processing...");
        } catch (Exception e) {
            logger.error("Work failed", e);
        }
    }
}

// Usage
LegacyLogger oldLogger = new LegacyLogger();
ModernLogger logger = new LegacyLoggerAdapter(oldLogger);
ModernService service = new ModernService(logger);
service.doWork();
```

---

## Adapter in Java Standard Library

### Arrays.asList()
```java
String[] array = {"a", "b", "c"};
List<String> list = Arrays.asList(array);  // Adapts array to List
```

### InputStreamReader
```java
// Adapts byte stream (InputStream) to character stream (Reader)
InputStream byteStream = new FileInputStream("file.txt");
Reader charStream = new InputStreamReader(byteStream);
```

### Collections.enumeration()
```java
// Adapts Iterator to Enumeration
List<String> list = Arrays.asList("a", "b", "c");
Enumeration<String> enumeration = Collections.enumeration(list);
```

---

## Two-Way Adapter

Sometimes you need adaptation in both directions:

```java
interface OldSystem {
    String oldMethod();
}

interface NewSystem {
    String newMethod();
}

class TwoWayAdapter implements OldSystem, NewSystem {
    private OldSystem oldSystem;
    private NewSystem newSystem;
    
    // Adapt old to new
    public TwoWayAdapter(OldSystem oldSystem) {
        this.oldSystem = oldSystem;
    }
    
    // Adapt new to old
    public TwoWayAdapter(NewSystem newSystem) {
        this.newSystem = newSystem;
    }
    
    @Override
    public String oldMethod() {
        if (oldSystem != null) {
            return oldSystem.oldMethod();
        }
        // Use newSystem's method to satisfy oldMethod
        return "Adapted from new: " + newSystem.newMethod();
    }
    
    @Override
    public String newMethod() {
        if (newSystem != null) {
            return newSystem.newMethod();
        }
        // Use oldSystem's method to satisfy newMethod
        return "Adapted from old: " + oldSystem.oldMethod();
    }
}
```

---

## When to Use Adapter Pattern

### ✅ Use When:
1. **Incompatible interfaces** need to work together
2. **Legacy code integration** with new code
3. **Third-party library** has different interface
4. **You can't modify** either side

### ❌ Don't Use When:
1. Interfaces are similar enough
2. You can modify the adaptee
3. Better to use a different pattern (like Facade)

---

## Adapter vs Other Patterns

| Pattern | Purpose |
|---------|---------|
| **Adapter** | Makes incompatible interfaces work together |
| **Facade** | Simplifies complex interface |
| **Decorator** | Adds functionality to existing interface |
| **Proxy** | Controls access to an object |

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Intent** | Convert one interface to another |
| **Key Idea** | Wrapper that translates calls |
| **Types** | Object Adapter (composition), Class Adapter (inheritance) |
| **Use When** | Integrating incompatible interfaces |

### Remember:
- Adapter wraps the adaptee
- Client works with target interface
- Adapter translates calls to adaptee
- Prefer object adapter (composition)

---

**Next: Decorator Pattern →**
