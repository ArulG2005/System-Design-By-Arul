# Abstract Factory Design Pattern

## Intent

> **Provide an interface for creating families of related or dependent objects without specifying their concrete classes.**

---

## The Problem

You need to create **families of related objects** that must work together:
- Windows UI components (Windows buttons, Windows menus, Windows dialogs)
- Mac UI components (Mac buttons, Mac menus, Mac dialogs)
- You can't mix Windows buttons with Mac dialogs!

---

## Simple Analogy

Think of a **Furniture Store**:
- They sell "Modern" style furniture: Modern Chair, Modern Table, Modern Sofa
- They also sell "Victorian" style: Victorian Chair, Victorian Table, Victorian Sofa
- You want your room to have **matching furniture** from ONE style
- Abstract Factory ensures you get a complete matching set

---

## Abstract Factory vs Factory Method

| Factory Method | Abstract Factory |
|----------------|------------------|
| Creates ONE product | Creates FAMILY of products |
| One factory method | Multiple factory methods |
| Uses inheritance | Uses composition |
| Single product variations | Multiple related products |

---

## Structure

```
┌──────────────────────────┐
│    AbstractFactory       │
├──────────────────────────┤
│ + createProductA(): A    │
│ + createProductB(): B    │
└───────────△──────────────┘
            │
   ┌────────┴────────┐
   │                 │
┌──────────────┐  ┌──────────────┐
│ConcreteFactory1│ │ConcreteFactory2│
├──────────────┤  ├──────────────┤
│+createProductA│  │+createProductA│
│+createProductB│  │+createProductB│
└──────────────┘  └──────────────┘
   │ creates          │ creates
   ▼                  ▼
┌──────────────┐  ┌──────────────┐
│ ProductA1    │  │ ProductA2    │
│ ProductB1    │  │ ProductB2    │
└──────────────┘  └──────────────┘
```

---

## Example 1: Cross-Platform UI Factory

```java
// ==================== ABSTRACT PRODUCTS ====================

interface Button {
    void render();
    void onClick();
}

interface Checkbox {
    void render();
    void toggle();
    boolean isChecked();
}

interface TextField {
    void render();
    void setText(String text);
    String getText();
}

// ==================== WINDOWS PRODUCTS ====================

class WindowsButton implements Button {
    @Override
    public void render() {
        System.out.println("[Windows Button]");
    }
    
    @Override
    public void onClick() {
        System.out.println("Windows button clicked!");
    }
}

class WindowsCheckbox implements Checkbox {
    private boolean checked = false;
    
    @Override
    public void render() {
        System.out.println("[" + (checked ? "X" : " ") + "] Windows Checkbox");
    }
    
    @Override
    public void toggle() {
        checked = !checked;
    }
    
    @Override
    public boolean isChecked() {
        return checked;
    }
}

class WindowsTextField implements TextField {
    private String text = "";
    
    @Override
    public void render() {
        System.out.println("[____" + text + "____] Windows TextField");
    }
    
    @Override
    public void setText(String text) {
        this.text = text;
    }
    
    @Override
    public String getText() {
        return text;
    }
}

// ==================== MAC PRODUCTS ====================

class MacButton implements Button {
    @Override
    public void render() {
        System.out.println("( Mac Button )");
    }
    
    @Override
    public void onClick() {
        System.out.println("Mac button clicked!");
    }
}

class MacCheckbox implements Checkbox {
    private boolean checked = false;
    
    @Override
    public void render() {
        System.out.println("(" + (checked ? "✓" : " ") + ") Mac Checkbox");
    }
    
    @Override
    public void toggle() {
        checked = !checked;
    }
    
    @Override
    public boolean isChecked() {
        return checked;
    }
}

class MacTextField implements TextField {
    private String text = "";
    
    @Override
    public void render() {
        System.out.println("| " + text + " | Mac TextField");
    }
    
    @Override
    public void setText(String text) {
        this.text = text;
    }
    
    @Override
    public String getText() {
        return text;
    }
}

// ==================== ABSTRACT FACTORY ====================

interface GUIFactory {
    Button createButton();
    Checkbox createCheckbox();
    TextField createTextField();
}

// ==================== CONCRETE FACTORIES ====================

class WindowsFactory implements GUIFactory {
    @Override
    public Button createButton() {
        return new WindowsButton();
    }
    
    @Override
    public Checkbox createCheckbox() {
        return new WindowsCheckbox();
    }
    
    @Override
    public TextField createTextField() {
        return new WindowsTextField();
    }
}

class MacFactory implements GUIFactory {
    @Override
    public Button createButton() {
        return new MacButton();
    }
    
    @Override
    public Checkbox createCheckbox() {
        return new MacCheckbox();
    }
    
    @Override
    public TextField createTextField() {
        return new MacTextField();
    }
}

// ==================== CLIENT CODE ====================

class Application {
    private Button button;
    private Checkbox checkbox;
    private TextField textField;
    
    public Application(GUIFactory factory) {
        // Create all components from the SAME factory
        button = factory.createButton();
        checkbox = factory.createCheckbox();
        textField = factory.createTextField();
    }
    
    public void render() {
        button.render();
        checkbox.render();
        textField.render();
    }
}

// Usage
public class Main {
    public static void main(String[] args) {
        String os = System.getProperty("os.name").toLowerCase();
        GUIFactory factory;
        
        if (os.contains("windows")) {
            factory = new WindowsFactory();
        } else if (os.contains("mac")) {
            factory = new MacFactory();
        } else {
            factory = new WindowsFactory(); // Default
        }
        
        Application app = new Application(factory);
        app.render();
    }
}
```

---

## Example 2: Furniture Factory

```java
// ==================== ABSTRACT PRODUCTS ====================

interface Chair {
    void sitOn();
    String getStyle();
}

interface Table {
    void putItem(String item);
    int getLegs();
    String getStyle();
}

interface Sofa {
    void layOn();
    int getSeats();
    String getStyle();
}

// ==================== MODERN FURNITURE ====================

class ModernChair implements Chair {
    @Override
    public void sitOn() {
        System.out.println("Sitting on sleek modern chair");
    }
    
    @Override
    public String getStyle() {
        return "Modern";
    }
}

class ModernTable implements Table {
    @Override
    public void putItem(String item) {
        System.out.println("Placing " + item + " on glass modern table");
    }
    
    @Override
    public int getLegs() {
        return 4;
    }
    
    @Override
    public String getStyle() {
        return "Modern";
    }
}

class ModernSofa implements Sofa {
    @Override
    public void layOn() {
        System.out.println("Relaxing on minimalist modern sofa");
    }
    
    @Override
    public int getSeats() {
        return 3;
    }
    
    @Override
    public String getStyle() {
        return "Modern";
    }
}

// ==================== VICTORIAN FURNITURE ====================

class VictorianChair implements Chair {
    @Override
    public void sitOn() {
        System.out.println("Sitting on ornate Victorian chair");
    }
    
    @Override
    public String getStyle() {
        return "Victorian";
    }
}

class VictorianTable implements Table {
    @Override
    public void putItem(String item) {
        System.out.println("Placing " + item + " on wooden Victorian table");
    }
    
    @Override
    public int getLegs() {
        return 6; // Ornate tables have more legs
    }
    
    @Override
    public String getStyle() {
        return "Victorian";
    }
}

class VictorianSofa implements Sofa {
    @Override
    public void layOn() {
        System.out.println("Relaxing on plush Victorian sofa");
    }
    
    @Override
    public int getSeats() {
        return 2; // Loveseats
    }
    
    @Override
    public String getStyle() {
        return "Victorian";
    }
}

// ==================== ART DECO FURNITURE ====================

class ArtDecoChair implements Chair {
    @Override
    public void sitOn() {
        System.out.println("Sitting on geometric Art Deco chair");
    }
    
    @Override
    public String getStyle() {
        return "Art Deco";
    }
}

class ArtDecoTable implements Table {
    @Override
    public void putItem(String item) {
        System.out.println("Placing " + item + " on lacquered Art Deco table");
    }
    
    @Override
    public int getLegs() {
        return 4;
    }
    
    @Override
    public String getStyle() {
        return "Art Deco";
    }
}

class ArtDecoSofa implements Sofa {
    @Override
    public void layOn() {
        System.out.println("Relaxing on velvet Art Deco sofa");
    }
    
    @Override
    public int getSeats() {
        return 4;
    }
    
    @Override
    public String getStyle() {
        return "Art Deco";
    }
}

// ==================== ABSTRACT FACTORY ====================

interface FurnitureFactory {
    Chair createChair();
    Table createTable();
    Sofa createSofa();
}

// ==================== CONCRETE FACTORIES ====================

class ModernFurnitureFactory implements FurnitureFactory {
    @Override
    public Chair createChair() {
        return new ModernChair();
    }
    
    @Override
    public Table createTable() {
        return new ModernTable();
    }
    
    @Override
    public Sofa createSofa() {
        return new ModernSofa();
    }
}

class VictorianFurnitureFactory implements FurnitureFactory {
    @Override
    public Chair createChair() {
        return new VictorianChair();
    }
    
    @Override
    public Table createTable() {
        return new VictorianTable();
    }
    
    @Override
    public Sofa createSofa() {
        return new VictorianSofa();
    }
}

class ArtDecoFurnitureFactory implements FurnitureFactory {
    @Override
    public Chair createChair() {
        return new ArtDecoChair();
    }
    
    @Override
    public Table createTable() {
        return new ArtDecoTable();
    }
    
    @Override
    public Sofa createSofa() {
        return new ArtDecoSofa();
    }
}

// ==================== CLIENT ====================

class Room {
    private Chair chair;
    private Table table;
    private Sofa sofa;
    
    public Room(FurnitureFactory factory) {
        chair = factory.createChair();
        table = factory.createTable();
        sofa = factory.createSofa();
    }
    
    public void describe() {
        System.out.println("Room decorated in " + chair.getStyle() + " style:");
        chair.sitOn();
        table.putItem("vase");
        sofa.layOn();
    }
}

// Usage
FurnitureFactory factory = new ModernFurnitureFactory();
Room livingRoom = new Room(factory);
livingRoom.describe();

// Easy to switch styles
FurnitureFactory victorianFactory = new VictorianFurnitureFactory();
Room studyRoom = new Room(victorianFactory);
studyRoom.describe();
```

---

## Example 3: Database Connection Factory

```java
// ==================== ABSTRACT PRODUCTS ====================

interface Connection {
    void connect();
    void disconnect();
    void executeQuery(String query);
}

interface Command {
    void execute();
    void undo();
}

interface Transaction {
    void begin();
    void commit();
    void rollback();
}

// ==================== MYSQL PRODUCTS ====================

class MySQLConnection implements Connection {
    @Override
    public void connect() {
        System.out.println("Connecting to MySQL database...");
    }
    
    @Override
    public void disconnect() {
        System.out.println("Disconnecting from MySQL...");
    }
    
    @Override
    public void executeQuery(String query) {
        System.out.println("MySQL executing: " + query);
    }
}

class MySQLCommand implements Command {
    private String sql;
    
    public MySQLCommand(String sql) {
        this.sql = sql;
    }
    
    @Override
    public void execute() {
        System.out.println("MySQL Command: " + sql);
    }
    
    @Override
    public void undo() {
        System.out.println("MySQL Rollback command");
    }
}

class MySQLTransaction implements Transaction {
    @Override
    public void begin() {
        System.out.println("START TRANSACTION;");
    }
    
    @Override
    public void commit() {
        System.out.println("COMMIT;");
    }
    
    @Override
    public void rollback() {
        System.out.println("ROLLBACK;");
    }
}

// ==================== POSTGRESQL PRODUCTS ====================

class PostgreSQLConnection implements Connection {
    @Override
    public void connect() {
        System.out.println("Connecting to PostgreSQL database...");
    }
    
    @Override
    public void disconnect() {
        System.out.println("Disconnecting from PostgreSQL...");
    }
    
    @Override
    public void executeQuery(String query) {
        System.out.println("PostgreSQL executing: " + query);
    }
}

class PostgreSQLCommand implements Command {
    private String sql;
    
    public PostgreSQLCommand(String sql) {
        this.sql = sql;
    }
    
    @Override
    public void execute() {
        System.out.println("PostgreSQL Command: " + sql);
    }
    
    @Override
    public void undo() {
        System.out.println("PostgreSQL Rollback command");
    }
}

class PostgreSQLTransaction implements Transaction {
    @Override
    public void begin() {
        System.out.println("BEGIN TRANSACTION;");
    }
    
    @Override
    public void commit() {
        System.out.println("END TRANSACTION;");
    }
    
    @Override
    public void rollback() {
        System.out.println("ABORT;");
    }
}

// ==================== ABSTRACT FACTORY ====================

interface DatabaseFactory {
    Connection createConnection();
    Command createCommand(String sql);
    Transaction createTransaction();
}

// ==================== CONCRETE FACTORIES ====================

class MySQLFactory implements DatabaseFactory {
    @Override
    public Connection createConnection() {
        return new MySQLConnection();
    }
    
    @Override
    public Command createCommand(String sql) {
        return new MySQLCommand(sql);
    }
    
    @Override
    public Transaction createTransaction() {
        return new MySQLTransaction();
    }
}

class PostgreSQLFactory implements DatabaseFactory {
    @Override
    public Connection createConnection() {
        return new PostgreSQLConnection();
    }
    
    @Override
    public Command createCommand(String sql) {
        return new PostgreSQLCommand(sql);
    }
    
    @Override
    public Transaction createTransaction() {
        return new PostgreSQLTransaction();
    }
}

// ==================== CLIENT ====================

class DatabaseClient {
    private Connection connection;
    private Transaction transaction;
    private DatabaseFactory factory;
    
    public DatabaseClient(DatabaseFactory factory) {
        this.factory = factory;
        this.connection = factory.createConnection();
        this.transaction = factory.createTransaction();
    }
    
    public void executeInTransaction(String... queries) {
        connection.connect();
        transaction.begin();
        
        try {
            for (String query : queries) {
                Command cmd = factory.createCommand(query);
                cmd.execute();
            }
            transaction.commit();
        } catch (Exception e) {
            transaction.rollback();
        } finally {
            connection.disconnect();
        }
    }
}

// Usage
DatabaseFactory factory = new MySQLFactory();
DatabaseClient client = new DatabaseClient(factory);
client.executeInTransaction(
    "INSERT INTO users VALUES (1, 'John')",
    "UPDATE accounts SET balance = 500 WHERE user_id = 1"
);
```

---

## Abstract Factory in Real World

### Java AWT/Swing
```java
// Different look and feel factories
UIManager.setLookAndFeel("javax.swing.plaf.metal.MetalLookAndFeel");
// or
UIManager.setLookAndFeel("com.sun.java.swing.plaf.windows.WindowsLookAndFeel");
```

### Java XML Parsers
```java
DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
// Returns platform-specific factory
```

---

## When to Use Abstract Factory

### ✅ Use When:
1. **Family of related products** that must work together
2. **Platform independence** (Windows/Mac/Linux)
3. **Product variations** but consistent interface
4. **Want to swap entire product families** at runtime

### ❌ Don't Use When:
1. Products are not related
2. Single product type
3. Simple object creation

---

## Pros and Cons

### Pros:
- Products from same factory are compatible
- Isolates concrete classes from client
- Easy to exchange product families
- Promotes consistency

### Cons:
- Adding new product types is difficult
- Many classes to maintain
- Can be complex for simple scenarios

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Intent** | Create families of related objects |
| **Key Idea** | One factory for one product family |
| **Components** | AbstractFactory, ConcreteFactories, AbstractProducts, ConcreteProducts |
| **Use When** | Related products that must work together |
| **Benefit** | Swap entire families easily |

---

**Next: Prototype Pattern →**
