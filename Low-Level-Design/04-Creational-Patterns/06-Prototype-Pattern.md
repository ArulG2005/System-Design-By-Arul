# Prototype Design Pattern

## Intent

> **Create new objects by cloning existing objects, avoiding the cost of creating from scratch.**

---

## The Problem

Object creation can be expensive when:
- Object requires complex initialization
- Object loads data from database/network
- Object requires many resource allocations
- Object creation involves complex calculations

Instead of creating new objects from scratch, **clone an existing one**!

---

## Simple Analogy

Think of **Cell Division (Mitosis)**:
- Creating a new cell from scratch is complex
- Instead, cells CLONE themselves
- The clone inherits all the properties of the original
- Each clone can then be modified independently

Or think of **Document Templates**:
- Instead of typing a letter from scratch every time
- You copy a template and modify it
- Much faster!

---

## Structure

```
┌──────────────────────────────┐
│      <<interface>>           │
│        Prototype             │
├──────────────────────────────┤
│ + clone(): Prototype         │
└──────────────────△───────────┘
                   │
          ┌────────┴────────┐
          │                 │
┌─────────────────┐  ┌─────────────────┐
│ConcretePrototype1│ │ConcretePrototype2│
├─────────────────┤  ├─────────────────┤
│ - field1        │  │ - field1        │
│ - field2        │  │ - field2        │
├─────────────────┤  ├─────────────────┤
│ + clone()       │  │ + clone()       │
└─────────────────┘  └─────────────────┘
```

---

## Basic Implementation

### Using Java's Cloneable Interface

```java
class Shape implements Cloneable {
    private int x;
    private int y;
    private String color;
    
    public Shape(int x, int y, String color) {
        this.x = x;
        this.y = y;
        this.color = color;
    }
    
    // Copy constructor
    public Shape(Shape source) {
        this.x = source.x;
        this.y = source.y;
        this.color = source.color;
    }
    
    @Override
    public Shape clone() {
        try {
            return (Shape) super.clone();  // Shallow copy
        } catch (CloneNotSupportedException e) {
            return new Shape(this);  // Fallback to copy constructor
        }
    }
    
    // Getters and Setters
    public int getX() { return x; }
    public void setX(int x) { this.x = x; }
    public int getY() { return y; }
    public void setY(int y) { this.y = y; }
    public String getColor() { return color; }
    public void setColor(String color) { this.color = color; }
    
    @Override
    public String toString() {
        return "Shape{x=" + x + ", y=" + y + ", color='" + color + "'}";
    }
}

// Usage
Shape original = new Shape(10, 20, "Red");
Shape clone = original.clone();

// Modify clone independently
clone.setX(100);
clone.setColor("Blue");

System.out.println(original);  // Shape{x=10, y=20, color='Red'}
System.out.println(clone);     // Shape{x=100, y=20, color='Blue'}
```

---

## Shallow Copy vs Deep Copy

### Shallow Copy
Copies object, but nested objects are **shared** (same reference).

```java
class Person implements Cloneable {
    private String name;
    private Address address;  // Nested object
    
    @Override
    public Person clone() {
        try {
            return (Person) super.clone();  // SHALLOW copy!
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException(e);
        }
    }
}

Person original = new Person("John", new Address("NYC"));
Person clone = original.clone();

// Both point to SAME Address object!
clone.getAddress().setCity("LA");
System.out.println(original.getAddress().getCity());  // LA (!!! Changed!)
```

### Deep Copy
Creates complete copies of all nested objects.

```java
class Person implements Cloneable {
    private String name;
    private Address address;
    
    public Person(String name, Address address) {
        this.name = name;
        this.address = address;
    }
    
    @Override
    public Person clone() {
        try {
            Person clone = (Person) super.clone();
            clone.address = this.address.clone();  // DEEP copy nested object!
            return clone;
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException(e);
        }
    }
}

class Address implements Cloneable {
    private String city;
    
    public Address(String city) {
        this.city = city;
    }
    
    @Override
    public Address clone() {
        try {
            return (Address) super.clone();
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException(e);
        }
    }
    
    public String getCity() { return city; }
    public void setCity(String city) { this.city = city; }
}

// Now deep copy works correctly
Person original = new Person("John", new Address("NYC"));
Person clone = original.clone();

clone.getAddress().setCity("LA");
System.out.println(original.getAddress().getCity());  // NYC (unchanged!)
System.out.println(clone.getAddress().getCity());     // LA
```

---

## Custom Prototype Interface

Better than relying on Java's Cloneable:

```java
interface Prototype<T> {
    T clone();
}

class Document implements Prototype<Document> {
    private String title;
    private String content;
    private List<String> attachments;
    
    public Document(String title, String content) {
        this.title = title;
        this.content = content;
        this.attachments = new ArrayList<>();
    }
    
    // Copy constructor for deep copy
    private Document(Document source) {
        this.title = source.title;
        this.content = source.content;
        this.attachments = new ArrayList<>(source.attachments);  // Copy list
    }
    
    @Override
    public Document clone() {
        return new Document(this);
    }
    
    public void addAttachment(String attachment) {
        attachments.add(attachment);
    }
    
    @Override
    public String toString() {
        return "Document{title='" + title + "', content='" + content + 
               "', attachments=" + attachments + '}';
    }
    
    // Getters and setters...
}

// Usage
Document template = new Document("Weekly Report", "Standard report format...");
template.addAttachment("template_header.png");

Document monday = template.clone();
monday.setTitle("Monday Report");
monday.addAttachment("monday_data.xlsx");

Document friday = template.clone();
friday.setTitle("Friday Report");
friday.addAttachment("friday_data.xlsx");
```

---

## Real-World Examples

### Example 1: Game Character Cloning

```java
abstract class GameCharacter implements Cloneable {
    protected String name;
    protected int health;
    protected int attack;
    protected int defense;
    protected List<String> abilities;
    protected Map<String, Integer> stats;
    
    public GameCharacter() {
        this.abilities = new ArrayList<>();
        this.stats = new HashMap<>();
    }
    
    @Override
    public GameCharacter clone() {
        try {
            GameCharacter clone = (GameCharacter) super.clone();
            clone.abilities = new ArrayList<>(this.abilities);  // Deep copy
            clone.stats = new HashMap<>(this.stats);  // Deep copy
            return clone;
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException(e);
        }
    }
    
    public abstract void specialMove();
    
    // Getters and setters...
}

class Warrior extends GameCharacter {
    private String weaponType;
    
    public Warrior(String name, String weaponType) {
        this.name = name;
        this.weaponType = weaponType;
        this.health = 100;
        this.attack = 50;
        this.defense = 30;
        this.abilities.add("Slash");
        this.abilities.add("Shield Block");
    }
    
    @Override
    public Warrior clone() {
        Warrior clone = (Warrior) super.clone();
        clone.weaponType = this.weaponType;
        return clone;
    }
    
    @Override
    public void specialMove() {
        System.out.println(name + " performs BERSERKER RAGE with " + weaponType);
    }
}

class Mage extends GameCharacter {
    private int mana;
    
    public Mage(String name, int mana) {
        this.name = name;
        this.mana = mana;
        this.health = 60;
        this.attack = 70;
        this.defense = 10;
        this.abilities.add("Fireball");
        this.abilities.add("Ice Blast");
    }
    
    @Override
    public Mage clone() {
        Mage clone = (Mage) super.clone();
        clone.mana = this.mana;
        return clone;
    }
    
    @Override
    public void specialMove() {
        System.out.println(name + " casts METEOR SHOWER!");
    }
}

// Prototype Registry
class CharacterRegistry {
    private Map<String, GameCharacter> prototypes = new HashMap<>();
    
    public void registerPrototype(String key, GameCharacter prototype) {
        prototypes.put(key, prototype);
    }
    
    public GameCharacter create(String key) {
        GameCharacter prototype = prototypes.get(key);
        if (prototype == null) {
            throw new IllegalArgumentException("Prototype not found: " + key);
        }
        return prototype.clone();
    }
}

// Usage
CharacterRegistry registry = new CharacterRegistry();
registry.registerPrototype("warrior", new Warrior("Template Warrior", "Sword"));
registry.registerPrototype("mage", new Mage("Template Mage", 100));

// Create characters by cloning
GameCharacter player1 = registry.create("warrior");
player1.setName("Conan");

GameCharacter player2 = registry.create("mage");
player2.setName("Gandalf");

GameCharacter enemy1 = registry.create("warrior");
enemy1.setName("Evil Knight");
```

---

### Example 2: Configuration Templates

```java
class DatabaseConfig implements Cloneable {
    private String host;
    private int port;
    private String database;
    private String username;
    private String password;
    private Map<String, String> connectionParams;
    
    public DatabaseConfig(String host, int port, String database) {
        this.host = host;
        this.port = port;
        this.database = database;
        this.connectionParams = new HashMap<>();
        this.connectionParams.put("autoReconnect", "true");
        this.connectionParams.put("useSSL", "false");
    }
    
    @Override
    public DatabaseConfig clone() {
        try {
            DatabaseConfig clone = (DatabaseConfig) super.clone();
            clone.connectionParams = new HashMap<>(this.connectionParams);
            return clone;
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException(e);
        }
    }
    
    public void setCredentials(String username, String password) {
        this.username = username;
        this.password = password;
    }
    
    public void setParam(String key, String value) {
        connectionParams.put(key, value);
    }
    
    @Override
    public String toString() {
        return String.format("jdbc:mysql://%s:%d/%s?user=%s&%s",
            host, port, database, username,
            connectionParams.entrySet().stream()
                .map(e -> e.getKey() + "=" + e.getValue())
                .collect(Collectors.joining("&")));
    }
}

// Create template configurations
DatabaseConfig productionTemplate = new DatabaseConfig("prod-db.example.com", 3306, "app_prod");
productionTemplate.setParam("useSSL", "true");
productionTemplate.setParam("connectionTimeout", "30000");

DatabaseConfig developmentTemplate = new DatabaseConfig("localhost", 3306, "app_dev");
developmentTemplate.setParam("useSSL", "false");

// Clone for specific uses
DatabaseConfig userServiceConfig = productionTemplate.clone();
userServiceConfig.setCredentials("user_service", "secret123");
userServiceConfig.setParam("maxPoolSize", "20");

DatabaseConfig reportServiceConfig = productionTemplate.clone();
reportServiceConfig.setCredentials("report_service", "secret456");
reportServiceConfig.setParam("maxPoolSize", "5");
reportServiceConfig.setParam("readOnly", "true");
```

---

### Example 3: Spreadsheet Cell Cloning

```java
interface CellPrototype {
    CellPrototype clone();
    void render();
}

class Cell implements CellPrototype {
    private String value;
    private CellStyle style;
    private String formula;
    
    public Cell(String value, CellStyle style) {
        this.value = value;
        this.style = style;
    }
    
    @Override
    public Cell clone() {
        Cell clone = new Cell(this.value, this.style.clone());
        clone.formula = this.formula;
        return clone;
    }
    
    @Override
    public void render() {
        System.out.println("[" + value + "] Style: " + style);
    }
    
    public void setValue(String value) { this.value = value; }
    public void setFormula(String formula) { this.formula = formula; }
}

class CellStyle implements Cloneable {
    private String fontFamily;
    private int fontSize;
    private String backgroundColor;
    private String textColor;
    private boolean bold;
    
    public CellStyle(String fontFamily, int fontSize) {
        this.fontFamily = fontFamily;
        this.fontSize = fontSize;
        this.backgroundColor = "white";
        this.textColor = "black";
        this.bold = false;
    }
    
    @Override
    public CellStyle clone() {
        try {
            return (CellStyle) super.clone();
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException(e);
        }
    }
    
    public void setBold(boolean bold) { this.bold = bold; }
    public void setBackgroundColor(String color) { this.backgroundColor = color; }
    
    @Override
    public String toString() {
        return fontFamily + " " + fontSize + "pt" + (bold ? " Bold" : "") +
               " bg:" + backgroundColor;
    }
}

// Usage
CellStyle headerStyle = new CellStyle("Arial", 14);
headerStyle.setBold(true);
headerStyle.setBackgroundColor("gray");

Cell headerTemplate = new Cell("Header", headerStyle);

// Clone header cells
Cell col1Header = headerTemplate.clone();
col1Header.setValue("Name");

Cell col2Header = headerTemplate.clone();
col2Header.setValue("Age");

Cell col3Header = headerTemplate.clone();
col3Header.setValue("Email");

// All have same style, different values
col1Header.render();  // [Name] Style: Arial 14pt Bold bg:gray
col2Header.render();  // [Age] Style: Arial 14pt Bold bg:gray
col3Header.render();  // [Email] Style: Arial 14pt Bold bg:gray
```

---

## Prototype Registry

A central place to store and retrieve prototypes:

```java
class PrototypeRegistry {
    private Map<String, Prototype<?>> prototypes = new HashMap<>();
    
    public void register(String key, Prototype<?> prototype) {
        prototypes.put(key, prototype);
    }
    
    public void unregister(String key) {
        prototypes.remove(key);
    }
    
    @SuppressWarnings("unchecked")
    public <T extends Prototype<T>> T get(String key) {
        Prototype<?> prototype = prototypes.get(key);
        if (prototype == null) {
            throw new IllegalArgumentException("Prototype not found: " + key);
        }
        return (T) prototype.clone();
    }
    
    public boolean contains(String key) {
        return prototypes.containsKey(key);
    }
}
```

---

## Prototype in Java Standard Library

### Object.clone()
```java
Object clone = obj.clone();  // Built-in cloning
```

### Array Copy
```java
int[] original = {1, 2, 3, 4, 5};
int[] clone = original.clone();
```

### Collections
```java
ArrayList<String> original = new ArrayList<>();
ArrayList<String> clone = (ArrayList<String>) original.clone();

// Or using copy constructor
ArrayList<String> clone2 = new ArrayList<>(original);
```

---

## When to Use Prototype Pattern

### ✅ Use When:
1. **Object creation is expensive** (database load, network, calculations)
2. **Many variations of similar objects** needed
3. **Runtime configuration** determines object types
4. **Avoiding complex class hierarchies** for factories

### ❌ Don't Use When:
1. Objects are simple to create
2. Objects have no state to copy
3. Deep copying is complex (circular references)

---

## Pros and Cons

### Pros:
- Avoids expensive object creation
- Add/remove prototypes at runtime
- Reduces subclasses
- Configure complex objects dynamically

### Cons:
- Cloning complex objects with circular references is difficult
- Must implement clone properly (deep vs shallow)
- Some objects may be impossible to clone

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Intent** | Clone existing objects instead of creating new ones |
| **Key Idea** | Copy constructor or clone method |
| **Types** | Shallow copy (shared references) vs Deep copy (all new) |
| **Registry** | Central storage for prototype objects |
| **Use When** | Object creation is expensive |

### Remember:
- Implement proper deep copy for nested objects
- Use prototype registry for managing prototypes
- Consider copy constructors as alternative to clone()
- Be careful with circular references

---

**Next: Adapter Pattern (Structural Patterns) →**
