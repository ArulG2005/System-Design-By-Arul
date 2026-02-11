# Template Method Design Pattern

## Intent

> **Define the skeleton of an algorithm in an operation, deferring some steps to subclasses. Template Method lets subclasses redefine certain steps of an algorithm without changing the algorithm's structure.**

---

## The Problem

You have an algorithm with **fixed structure** but:
- Some steps **vary** across implementations
- You want to **avoid code duplication**
- Subclasses should customize parts, not the whole

---

## Simple Analogy

Think of **Building a House**:
1. Build foundation ← Same for all houses
2. Build walls ← Same structure, different materials
3. Build roof ← Different styles (flat, peaked, etc.)
4. Add windows/doors ← Same process
5. Interior finish ← Different designs

The **template** (algorithm) is fixed:
- Foundation → Walls → Roof → Windows → Interior

But **specific steps** can be customized!

---

## Structure

```
┌──────────────────────────────────────────────────┐
│              AbstractClass                       │
├──────────────────────────────────────────────────┤
│ + templateMethod() {                             │
│     step1();                                     │
│     step2();  // abstract                        │
│     step3();                                     │
│     step4();  // abstract                        │
│   }                                              │
│ + step1() { ... }        // concrete             │
│ + step2()                // abstract             │
│ + step3() { ... }        // concrete             │
│ + step4()                // abstract             │
│ + hook() { }             // optional hook        │
└──────────────────────────△───────────────────────┘
                           │
          ┌────────────────┴────────────────┐
          │                                 │
┌─────────────────────────┐   ┌─────────────────────────┐
│   ConcreteClassA        │   │   ConcreteClassB        │
├─────────────────────────┤   ├─────────────────────────┤
│ + step2() { ... }       │   │ + step2() { ... }       │
│ + step4() { ... }       │   │ + step4() { ... }       │
└─────────────────────────┘   └─────────────────────────┘
```

---

## Basic Example: Beverage Recipe

```java
// Abstract class with template method
abstract class Beverage {
    
    // TEMPLATE METHOD - final so subclasses can't change the algorithm
    public final void prepare() {
        boilWater();
        brew();           // abstract - subclasses implement
        pourInCup();
        if (customerWantsCondiments()) {  // hook
            addCondiments();  // abstract
        }
    }
    
    // Concrete methods (same for all)
    private void boilWater() {
        System.out.println("Boiling water");
    }
    
    private void pourInCup() {
        System.out.println("Pouring into cup");
    }
    
    // Abstract methods (subclasses must implement)
    protected abstract void brew();
    protected abstract void addCondiments();
    
    // Hook method (optional override)
    protected boolean customerWantsCondiments() {
        return true;  // default
    }
}

// Concrete implementation: Coffee
class Coffee extends Beverage {
    
    @Override
    protected void brew() {
        System.out.println("Dripping coffee through filter");
    }
    
    @Override
    protected void addCondiments() {
        System.out.println("Adding sugar and milk");
    }
}

// Concrete implementation: Tea
class Tea extends Beverage {
    
    @Override
    protected void brew() {
        System.out.println("Steeping the tea");
    }
    
    @Override
    protected void addCondiments() {
        System.out.println("Adding lemon");
    }
    
    // Override hook
    @Override
    protected boolean customerWantsCondiments() {
        // Could ask user or check preference
        return false;  // No condiments for tea
    }
}

// Concrete implementation: Hot Chocolate
class HotChocolate extends Beverage {
    
    @Override
    protected void brew() {
        System.out.println("Mixing cocoa powder with hot water");
    }
    
    @Override
    protected void addCondiments() {
        System.out.println("Adding whipped cream and marshmallows");
    }
}

// Usage
public class BeverageShop {
    public static void main(String[] args) {
        System.out.println("=== Making Coffee ===");
        Beverage coffee = new Coffee();
        coffee.prepare();
        
        System.out.println("\n=== Making Tea ===");
        Beverage tea = new Tea();
        tea.prepare();
        
        System.out.println("\n=== Making Hot Chocolate ===");
        Beverage hotChocolate = new HotChocolate();
        hotChocolate.prepare();
    }
}

/* Output:
=== Making Coffee ===
Boiling water
Dripping coffee through filter
Pouring into cup
Adding sugar and milk

=== Making Tea ===
Boiling water
Steeping the tea
Pouring into cup

=== Making Hot Chocolate ===
Boiling water
Mixing cocoa powder with hot water
Pouring into cup
Adding whipped cream and marshmallows
*/
```

---

## Real-World Examples

### Example 1: Data Processing Pipeline

```java
// Abstract data processor
abstract class DataProcessor {
    
    // Template method
    public final void process() {
        readData();
        processData();
        if (shouldValidate()) {  // hook
            validateData();
        }
        writeData();
        cleanup();
    }
    
    // Abstract methods - must be implemented
    protected abstract void readData();
    protected abstract void processData();
    protected abstract void writeData();
    
    // Concrete method with default implementation
    protected void validateData() {
        System.out.println("Validating data...");
    }
    
    protected void cleanup() {
        System.out.println("Cleaning up resources\n");
    }
    
    // Hook - subclasses can override
    protected boolean shouldValidate() {
        return true;
    }
}

// CSV Processor
class CSVProcessor extends DataProcessor {
    private List<String[]> data;
    
    @Override
    protected void readData() {
        System.out.println("Reading data from CSV file");
        data = new ArrayList<>();
        data.add(new String[]{"John", "30", "Developer"});
        data.add(new String[]{"Jane", "25", "Designer"});
    }
    
    @Override
    protected void processData() {
        System.out.println("Processing CSV data - converting to uppercase");
        for (String[] row : data) {
            for (int i = 0; i < row.length; i++) {
                row[i] = row[i].toUpperCase();
            }
        }
    }
    
    @Override
    protected void writeData() {
        System.out.println("Writing processed data to output.csv");
        for (String[] row : data) {
            System.out.println("  " + String.join(",", row));
        }
    }
}

// Database Processor
class DatabaseProcessor extends DataProcessor {
    private List<Map<String, Object>> records;
    
    @Override
    protected void readData() {
        System.out.println("Reading data from database");
        records = new ArrayList<>();
        records.add(Map.of("id", 1, "name", "Product A", "price", 99.99));
        records.add(Map.of("id", 2, "name", "Product B", "price", 149.99));
    }
    
    @Override
    protected void processData() {
        System.out.println("Processing database records - calculating totals");
    }
    
    @Override
    protected void writeData() {
        System.out.println("Writing results back to database");
        for (Map<String, Object> record : records) {
            System.out.println("  Updated: " + record);
        }
    }
    
    @Override
    protected boolean shouldValidate() {
        return false;  // Skip validation for DB
    }
}

// API Processor
class APIProcessor extends DataProcessor {
    private String jsonData;
    
    @Override
    protected void readData() {
        System.out.println("Fetching data from REST API");
        jsonData = "{\"users\": [{\"id\": 1}, {\"id\": 2}]}";
    }
    
    @Override
    protected void processData() {
        System.out.println("Parsing and transforming JSON data");
    }
    
    @Override
    protected void writeData() {
        System.out.println("Sending processed data to another API");
    }
    
    @Override
    protected void validateData() {
        System.out.println("Validating JSON schema");
    }
}

// Usage
public class DataPipelineApp {
    public static void main(String[] args) {
        System.out.println("=== CSV Processing ===");
        DataProcessor csvProcessor = new CSVProcessor();
        csvProcessor.process();
        
        System.out.println("=== Database Processing ===");
        DataProcessor dbProcessor = new DatabaseProcessor();
        dbProcessor.process();
        
        System.out.println("=== API Processing ===");
        DataProcessor apiProcessor = new APIProcessor();
        apiProcessor.process();
    }
}
```

---

### Example 2: Game AI

```java
// Abstract game character
abstract class GameCharacter {
    
    // Template method for turn
    public final void takeTurn() {
        System.out.println("\n--- " + getName() + "'s Turn ---");
        
        beforeTurn();    // hook
        move();
        attack();
        useSpecialAbility();
        afterTurn();     // hook
    }
    
    // Abstract methods
    protected abstract String getName();
    protected abstract void move();
    protected abstract void attack();
    protected abstract void useSpecialAbility();
    
    // Hooks with default (empty) implementation
    protected void beforeTurn() { }
    protected void afterTurn() { }
}

// Warrior
class Warrior extends GameCharacter {
    private int rage = 0;
    
    @Override
    protected String getName() {
        return "Warrior";
    }
    
    @Override
    protected void move() {
        System.out.println("Moving 3 spaces (heavy armor)");
    }
    
    @Override
    protected void attack() {
        System.out.println("Swinging sword for 25 damage");
        rage += 10;
    }
    
    @Override
    protected void useSpecialAbility() {
        if (rage >= 30) {
            System.out.println("BERSERKER RAGE! Double damage next attack!");
            rage = 0;
        } else {
            System.out.println("Building rage: " + rage + "/30");
        }
    }
    
    @Override
    protected void afterTurn() {
        System.out.println("Raising shield for defense");
    }
}

// Mage
class Mage extends GameCharacter {
    private int mana = 100;
    
    @Override
    protected String getName() {
        return "Mage";
    }
    
    @Override
    protected void move() {
        System.out.println("Teleporting 5 spaces");
        mana -= 10;
    }
    
    @Override
    protected void attack() {
        System.out.println("Casting fireball for 40 damage");
        mana -= 20;
    }
    
    @Override
    protected void useSpecialAbility() {
        if (mana >= 50) {
            System.out.println("Casting METEOR STORM! Area damage!");
            mana -= 50;
        } else {
            System.out.println("Not enough mana. Regenerating...");
            mana += 20;
        }
    }
    
    @Override
    protected void beforeTurn() {
        System.out.println("Mana: " + mana);
    }
}

// Rogue
class Rogue extends GameCharacter {
    private boolean isStealthed = false;
    
    @Override
    protected String getName() {
        return "Rogue";
    }
    
    @Override
    protected void move() {
        System.out.println("Dashing 6 spaces (fast and silent)");
    }
    
    @Override
    protected void attack() {
        if (isStealthed) {
            System.out.println("BACKSTAB! Critical hit for 60 damage!");
            isStealthed = false;
        } else {
            System.out.println("Quick strike for 20 damage");
        }
    }
    
    @Override
    protected void useSpecialAbility() {
        if (!isStealthed) {
            System.out.println("Entering STEALTH mode");
            isStealthed = true;
        }
    }
}

// Usage
public class GameDemo {
    public static void main(String[] args) {
        List<GameCharacter> party = Arrays.asList(
            new Warrior(),
            new Mage(),
            new Rogue()
        );
        
        // Simulate 2 rounds
        for (int round = 1; round <= 2; round++) {
            System.out.println("\n========== ROUND " + round + " ==========");
            for (GameCharacter character : party) {
                character.takeTurn();
            }
        }
    }
}
```

---

### Example 3: Document Generator

```java
// Abstract document generator
abstract class DocumentGenerator {
    
    // Template method
    public final String generate(String title, List<String> sections) {
        StringBuilder doc = new StringBuilder();
        
        doc.append(createHeader(title));
        doc.append(createTableOfContents(sections));
        
        for (String section : sections) {
            doc.append(createSection(section));
        }
        
        if (includeFooter()) {
            doc.append(createFooter());
        }
        
        return doc.toString();
    }
    
    // Abstract methods
    protected abstract String createHeader(String title);
    protected abstract String createSection(String content);
    protected abstract String createFooter();
    
    // Concrete method with default implementation
    protected String createTableOfContents(List<String> sections) {
        StringBuilder toc = new StringBuilder("Contents:\n");
        for (int i = 0; i < sections.size(); i++) {
            toc.append("  ").append(i + 1).append(". Section ").append(i + 1).append("\n");
        }
        return toc.toString();
    }
    
    // Hook
    protected boolean includeFooter() {
        return true;
    }
}

// HTML Generator
class HTMLGenerator extends DocumentGenerator {
    
    @Override
    protected String createHeader(String title) {
        return "<html>\n<head><title>" + title + "</title></head>\n<body>\n" +
               "<h1>" + title + "</h1>\n";
    }
    
    @Override
    protected String createSection(String content) {
        return "<section>\n<p>" + content + "</p>\n</section>\n";
    }
    
    @Override
    protected String createFooter() {
        return "<footer>Generated by DocumentGenerator</footer>\n</body>\n</html>";
    }
    
    @Override
    protected String createTableOfContents(List<String> sections) {
        StringBuilder toc = new StringBuilder("<nav><ul>\n");
        for (int i = 0; i < sections.size(); i++) {
            toc.append("<li>Section ").append(i + 1).append("</li>\n");
        }
        toc.append("</ul></nav>\n");
        return toc.toString();
    }
}

// Markdown Generator
class MarkdownGenerator extends DocumentGenerator {
    
    @Override
    protected String createHeader(String title) {
        return "# " + title + "\n\n";
    }
    
    @Override
    protected String createSection(String content) {
        return "## Section\n\n" + content + "\n\n";
    }
    
    @Override
    protected String createFooter() {
        return "---\n*Generated by DocumentGenerator*\n";
    }
}

// Plain Text Generator
class PlainTextGenerator extends DocumentGenerator {
    
    @Override
    protected String createHeader(String title) {
        String border = "=".repeat(title.length() + 4);
        return border + "\n  " + title + "\n" + border + "\n\n";
    }
    
    @Override
    protected String createSection(String content) {
        return "* " + content + "\n\n";
    }
    
    @Override
    protected String createFooter() {
        return "\n--- End of Document ---\n";
    }
    
    @Override
    protected boolean includeFooter() {
        return false;  // No footer for plain text
    }
}

// Usage
public class DocumentApp {
    public static void main(String[] args) {
        List<String> sections = Arrays.asList(
            "Introduction to the topic",
            "Main content goes here",
            "Conclusion and summary"
        );
        
        System.out.println("=== HTML Document ===");
        DocumentGenerator htmlGen = new HTMLGenerator();
        System.out.println(htmlGen.generate("My Document", sections));
        
        System.out.println("\n=== Markdown Document ===");
        DocumentGenerator mdGen = new MarkdownGenerator();
        System.out.println(mdGen.generate("My Document", sections));
        
        System.out.println("\n=== Plain Text Document ===");
        DocumentGenerator txtGen = new PlainTextGenerator();
        System.out.println(txtGen.generate("My Document", sections));
    }
}
```

---

## Understanding Hooks

```java
abstract class Template {
    
    public final void execute() {
        step1();
        if (hookShouldDoStep2()) {  // HOOK - optional override
            step2();
        }
        step3();
        hookAfterStep3();  // HOOK - optional extension point
    }
    
    // Required step - subclass MUST implement
    protected abstract void step1();
    protected abstract void step3();
    
    // Optional step
    protected void step2() {
        System.out.println("Default step 2");
    }
    
    // Hooks - empty by default, subclass MAY override
    protected boolean hookShouldDoStep2() {
        return true;  // Default: do step 2
    }
    
    protected void hookAfterStep3() {
        // Default: do nothing
    }
}
```

---

## Hollywood Principle

> **"Don't call us, we'll call you"**

- Parent class (template) **calls** subclass methods
- Subclass doesn't control the flow
- Framework calls your code, not vice versa

```java
// Framework controls the flow
abstract class Framework {
    public final void run() {
        setup();       // Framework calls
        doWork();      // Framework calls your implementation
        teardown();    // Framework calls
    }
    
    protected abstract void doWork();  // You implement, framework calls
}
```

---

## When to Use Template Method

### ✅ Use When:
1. Multiple classes share **same algorithm structure**
2. Want to **avoid code duplication**
3. Need to control which parts can be **customized**
4. Subclasses should customize **steps, not structure**

### ❌ Don't Use When:
1. Algorithm varies significantly between implementations
2. Only one or two implementations exist
3. Flexibility of runtime behavior switching is needed (use Strategy)

---

## Template Method vs Strategy

| Template Method | Strategy |
|-----------------|----------|
| Uses **inheritance** | Uses **composition** |
| Algorithm skeleton **fixed** | Entire algorithm **varies** |
| Compile-time binding | Runtime binding |
| Subclass customizes **steps** | Strategy replaces **whole algorithm** |

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Intent** | Define algorithm skeleton, defer steps to subclasses |
| **Key Idea** | Fixed structure, variable steps |
| **Benefits** | Code reuse, controlled customization |
| **Components** | Template method, abstract steps, hooks |

### Remember:
- Template method should be **final** (prevent override)
- Abstract methods = **required** customization points
- Hooks = **optional** customization points
- Follows **Hollywood Principle**: framework calls subclass

---

**Next: State Pattern →**
