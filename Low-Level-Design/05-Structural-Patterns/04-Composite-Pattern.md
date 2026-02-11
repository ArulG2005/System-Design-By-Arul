# Composite Design Pattern

## Intent

> **Compose objects into tree structures to represent part-whole hierarchies. Composite lets clients treat individual objects and compositions of objects uniformly.**

---

## The Problem

You have a **tree-like structure**:
- Items can contain other items (folders contain files)
- You want to treat individual items and groups the same way
- Don't want to check "is this a single item or a group?" everywhere

---

## Simple Analogy

Think of a **File System**:
- A **File** is a leaf (can't contain other items)
- A **Folder** is a composite (contains files or other folders)
- Both can be "opened", "deleted", "copied"
- You treat them the same way!

Or think of a **Company Hierarchy**:
- An **Employee** is a leaf
- A **Manager** is a composite (has employees under them)
- Both have salary, name, work()
- Calculate total salary of a department = sum of all employees

---

## Structure

```
┌─────────────────────────────────────┐
│            Component                │◄────────────────┐
├─────────────────────────────────────┤                 │
│ + operation()                       │                 │
│ + add(Component)                    │                 │
│ + remove(Component)                 │                 │
│ + getChild(int)                     │                 │
└────────────────△────────────────────┘                 │
                 │                                      │ children
        ┌────────┴────────┐                            │
        │                 │                            │
┌───────────────┐  ┌──────────────────────────┐       │
│     Leaf      │  │       Composite          │───────┘
├───────────────┤  ├──────────────────────────┤
│ + operation() │  │ - children: Component[]  │
└───────────────┘  │ + operation()            │
                   │ + add(Component)         │
                   │ + remove(Component)      │
                   │ + getChild(int)          │
                   └──────────────────────────┘
```

---

## Basic Example: File System

```java
// Component
interface FileSystemItem {
    String getName();
    int getSize();  // in KB
    void display(String indent);
}

// Leaf
class File implements FileSystemItem {
    private String name;
    private int size;
    
    public File(String name, int size) {
        this.name = name;
        this.size = size;
    }
    
    @Override
    public String getName() {
        return name;
    }
    
    @Override
    public int getSize() {
        return size;
    }
    
    @Override
    public void display(String indent) {
        System.out.println(indent + "📄 " + name + " (" + size + " KB)");
    }
}

// Composite
class Folder implements FileSystemItem {
    private String name;
    private List<FileSystemItem> children = new ArrayList<>();
    
    public Folder(String name) {
        this.name = name;
    }
    
    public void add(FileSystemItem item) {
        children.add(item);
    }
    
    public void remove(FileSystemItem item) {
        children.remove(item);
    }
    
    @Override
    public String getName() {
        return name;
    }
    
    @Override
    public int getSize() {
        int total = 0;
        for (FileSystemItem child : children) {
            total += child.getSize();  // Works for both files AND folders!
        }
        return total;
    }
    
    @Override
    public void display(String indent) {
        System.out.println(indent + "📁 " + name + " (" + getSize() + " KB)");
        for (FileSystemItem child : children) {
            child.display(indent + "  ");
        }
    }
}

// Usage
public class Main {
    public static void main(String[] args) {
        // Create files
        File file1 = new File("document.txt", 10);
        File file2 = new File("image.png", 200);
        File file3 = new File("music.mp3", 5000);
        File file4 = new File("video.mp4", 50000);
        
        // Create folder structure
        Folder documents = new Folder("Documents");
        documents.add(file1);
        
        Folder media = new Folder("Media");
        media.add(file2);
        media.add(file3);
        media.add(file4);
        
        Folder root = new Folder("Root");
        root.add(documents);
        root.add(media);
        
        // Display entire tree
        root.display("");
        
        // Get total size - uniform treatment!
        System.out.println("\nTotal size: " + root.getSize() + " KB");
    }
}

/* Output:
📁 Root (55210 KB)
  📁 Documents (10 KB)
    📄 document.txt (10 KB)
  📁 Media (55200 KB)
    📄 image.png (200 KB)
    📄 music.mp3 (5000 KB)
    📄 video.mp4 (50000 KB)

Total size: 55210 KB
*/
```

---

## Real-World Examples

### Example 1: Organization Hierarchy

```java
// Component
interface Employee {
    String getName();
    double getSalary();
    void showDetails(String indent);
}

// Leaf - Individual contributor
class Developer implements Employee {
    private String name;
    private double salary;
    
    public Developer(String name, double salary) {
        this.name = name;
        this.salary = salary;
    }
    
    @Override
    public String getName() {
        return name;
    }
    
    @Override
    public double getSalary() {
        return salary;
    }
    
    @Override
    public void showDetails(String indent) {
        System.out.println(indent + "👨‍💻 " + name + " (Developer) - $" + salary);
    }
}

class Designer implements Employee {
    private String name;
    private double salary;
    
    public Designer(String name, double salary) {
        this.name = name;
        this.salary = salary;
    }
    
    @Override
    public String getName() {
        return name;
    }
    
    @Override
    public double getSalary() {
        return salary;
    }
    
    @Override
    public void showDetails(String indent) {
        System.out.println(indent + "🎨 " + name + " (Designer) - $" + salary);
    }
}

// Composite - Manager with team
class Manager implements Employee {
    private String name;
    private double salary;
    private List<Employee> team = new ArrayList<>();
    
    public Manager(String name, double salary) {
        this.name = name;
        this.salary = salary;
    }
    
    public void addEmployee(Employee employee) {
        team.add(employee);
    }
    
    public void removeEmployee(Employee employee) {
        team.remove(employee);
    }
    
    @Override
    public String getName() {
        return name;
    }
    
    @Override
    public double getSalary() {
        // Manager's own salary + entire team's salary
        double total = salary;
        for (Employee emp : team) {
            total += emp.getSalary();
        }
        return total;
    }
    
    @Override
    public void showDetails(String indent) {
        System.out.println(indent + "👔 " + name + " (Manager) - $" + salary);
        for (Employee emp : team) {
            emp.showDetails(indent + "  ");
        }
    }
}

// Usage
public class Organization {
    public static void main(String[] args) {
        // Create developers
        Developer dev1 = new Developer("Alice", 80000);
        Developer dev2 = new Developer("Bob", 85000);
        Developer dev3 = new Developer("Charlie", 90000);
        
        // Create designers
        Designer des1 = new Designer("Diana", 75000);
        Designer des2 = new Designer("Eve", 78000);
        
        // Create team leads
        Manager teamLead1 = new Manager("Frank", 120000);
        teamLead1.addEmployee(dev1);
        teamLead1.addEmployee(dev2);
        
        Manager teamLead2 = new Manager("Grace", 115000);
        teamLead2.addEmployee(dev3);
        teamLead2.addEmployee(des1);
        
        // Create department head
        Manager departmentHead = new Manager("Henry", 180000);
        departmentHead.addEmployee(teamLead1);
        departmentHead.addEmployee(teamLead2);
        departmentHead.addEmployee(des2);
        
        // Display entire organization
        departmentHead.showDetails("");
        
        // Calculate total department cost
        System.out.println("\nTotal Department Cost: $" + departmentHead.getSalary());
    }
}

/* Output:
👔 Henry (Manager) - $180000
  👔 Frank (Manager) - $120000
    👨‍💻 Alice (Developer) - $80000
    👨‍💻 Bob (Developer) - $85000
  👔 Grace (Manager) - $115000
    👨‍💻 Charlie (Developer) - $90000
    🎨 Diana (Designer) - $75000
  🎨 Eve (Designer) - $78000

Total Department Cost: $823000
*/
```

---

### Example 2: GUI Components

```java
// Component
interface UIComponent {
    void render();
    void resize(int width, int height);
}

// Leaf components
class Button implements UIComponent {
    private String label;
    
    public Button(String label) {
        this.label = label;
    }
    
    @Override
    public void render() {
        System.out.println("  [Button: " + label + "]");
    }
    
    @Override
    public void resize(int width, int height) {
        System.out.println("  Button '" + label + "' resized");
    }
}

class TextBox implements UIComponent {
    private String placeholder;
    
    public TextBox(String placeholder) {
        this.placeholder = placeholder;
    }
    
    @Override
    public void render() {
        System.out.println("  [TextBox: " + placeholder + "]");
    }
    
    @Override
    public void resize(int width, int height) {
        System.out.println("  TextBox resized");
    }
}

class Label implements UIComponent {
    private String text;
    
    public Label(String text) {
        this.text = text;
    }
    
    @Override
    public void render() {
        System.out.println("  " + text);
    }
    
    @Override
    public void resize(int width, int height) {
        System.out.println("  Label resized");
    }
}

// Composite - Panel can contain other components
class Panel implements UIComponent {
    private String name;
    private List<UIComponent> children = new ArrayList<>();
    
    public Panel(String name) {
        this.name = name;
    }
    
    public void add(UIComponent component) {
        children.add(component);
    }
    
    public void remove(UIComponent component) {
        children.remove(component);
    }
    
    @Override
    public void render() {
        System.out.println("╔═══ " + name + " ═══╗");
        for (UIComponent child : children) {
            child.render();
        }
        System.out.println("╚" + "═".repeat(name.length() + 8) + "╝");
    }
    
    @Override
    public void resize(int width, int height) {
        System.out.println("Panel '" + name + "' resizing to " + width + "x" + height);
        for (UIComponent child : children) {
            child.resize(width - 20, height - 20);
        }
    }
}

// Usage
public class GUIApp {
    public static void main(String[] args) {
        // Create login form
        Panel loginForm = new Panel("Login Form");
        loginForm.add(new Label("Username:"));
        loginForm.add(new TextBox("Enter username"));
        loginForm.add(new Label("Password:"));
        loginForm.add(new TextBox("Enter password"));
        loginForm.add(new Button("Login"));
        loginForm.add(new Button("Forgot Password?"));
        
        // Create sidebar
        Panel sidebar = new Panel("Sidebar");
        sidebar.add(new Button("Home"));
        sidebar.add(new Button("Profile"));
        sidebar.add(new Button("Settings"));
        
        // Create main panel containing everything
        Panel mainWindow = new Panel("Main Window");
        mainWindow.add(sidebar);
        mainWindow.add(loginForm);
        
        // Render everything with single call!
        mainWindow.render();
        
        System.out.println("\n--- Resizing window ---\n");
        mainWindow.resize(800, 600);
    }
}
```

---

### Example 3: Menu System (Restaurant)

```java
// Component
interface MenuComponent {
    String getName();
    double getPrice();
    void print(String indent);
}

// Leaf - Single menu item
class MenuItem implements MenuComponent {
    private String name;
    private String description;
    private double price;
    
    public MenuItem(String name, String description, double price) {
        this.name = name;
        this.description = description;
        this.price = price;
    }
    
    @Override
    public String getName() {
        return name;
    }
    
    @Override
    public double getPrice() {
        return price;
    }
    
    @Override
    public void print(String indent) {
        System.out.println(indent + name + " - $" + String.format("%.2f", price));
        System.out.println(indent + "  " + description);
    }
}

// Composite - Menu category containing items or sub-menus
class Menu implements MenuComponent {
    private String name;
    private List<MenuComponent> items = new ArrayList<>();
    
    public Menu(String name) {
        this.name = name;
    }
    
    public void add(MenuComponent component) {
        items.add(component);
    }
    
    public void remove(MenuComponent component) {
        items.remove(component);
    }
    
    @Override
    public String getName() {
        return name;
    }
    
    @Override
    public double getPrice() {
        // Total of all items
        double total = 0;
        for (MenuComponent item : items) {
            total += item.getPrice();
        }
        return total;
    }
    
    @Override
    public void print(String indent) {
        System.out.println(indent + "=== " + name.toUpperCase() + " ===");
        for (MenuComponent item : items) {
            item.print(indent + "  ");
        }
    }
}

// Usage
public class Restaurant {
    public static void main(String[] args) {
        // Create appetizers menu
        Menu appetizers = new Menu("Appetizers");
        appetizers.add(new MenuItem("Spring Rolls", "Crispy vegetable rolls", 6.99));
        appetizers.add(new MenuItem("Soup of the Day", "Chef's special soup", 4.99));
        appetizers.add(new MenuItem("Salad", "Fresh garden salad", 5.99));
        
        // Create main course menu
        Menu mainCourse = new Menu("Main Course");
        mainCourse.add(new MenuItem("Grilled Salmon", "With lemon butter sauce", 18.99));
        mainCourse.add(new MenuItem("Steak", "8oz ribeye with sides", 24.99));
        mainCourse.add(new MenuItem("Pasta", "Creamy alfredo pasta", 14.99));
        
        // Create desserts menu
        Menu desserts = new Menu("Desserts");
        desserts.add(new MenuItem("Cheesecake", "New York style", 7.99));
        desserts.add(new MenuItem("Ice Cream", "Three scoops", 5.99));
        
        // Create beverages sub-menu
        Menu beverages = new Menu("Beverages");
        beverages.add(new MenuItem("Coffee", "Fresh brewed", 2.99));
        beverages.add(new MenuItem("Tea", "Assorted teas", 2.49));
        
        Menu alcoholic = new Menu("Alcoholic");
        alcoholic.add(new MenuItem("House Wine", "Glass of red or white", 8.99));
        alcoholic.add(new MenuItem("Beer", "Draft beer", 5.99));
        beverages.add(alcoholic);  // Sub-menu inside beverages
        
        // Create full menu
        Menu fullMenu = new Menu("Restaurant Menu");
        fullMenu.add(appetizers);
        fullMenu.add(mainCourse);
        fullMenu.add(desserts);
        fullMenu.add(beverages);
        
        // Print entire menu with single call!
        fullMenu.print("");
    }
}

/* Output:
=== RESTAURANT MENU ===
  === APPETIZERS ===
    Spring Rolls - $6.99
      Crispy vegetable rolls
    Soup of the Day - $4.99
      Chef's special soup
    Salad - $5.99
      Fresh garden salad
  === MAIN COURSE ===
    Grilled Salmon - $18.99
      With lemon butter sauce
    Steak - $24.99
      8oz ribeye with sides
    Pasta - $14.99
      Creamy alfredo pasta
  === DESSERTS ===
    Cheesecake - $7.99
      New York style
    Ice Cream - $5.99
      Three scoops
  === BEVERAGES ===
    Coffee - $2.99
      Fresh brewed
    Tea - $2.49
      Assorted teas
    === ALCOHOLIC ===
      House Wine - $8.99
        Glass of red or white
      Beer - $5.99
        Draft beer
*/
```

---

### Example 4: Expression Tree

```java
// Component
interface Expression {
    int evaluate();
    String print();
}

// Leaf - Numbers
class Number implements Expression {
    private int value;
    
    public Number(int value) {
        this.value = value;
    }
    
    @Override
    public int evaluate() {
        return value;
    }
    
    @Override
    public String print() {
        return String.valueOf(value);
    }
}

// Composite - Operations
class Addition implements Expression {
    private Expression left;
    private Expression right;
    
    public Addition(Expression left, Expression right) {
        this.left = left;
        this.right = right;
    }
    
    @Override
    public int evaluate() {
        return left.evaluate() + right.evaluate();
    }
    
    @Override
    public String print() {
        return "(" + left.print() + " + " + right.print() + ")";
    }
}

class Multiplication implements Expression {
    private Expression left;
    private Expression right;
    
    public Multiplication(Expression left, Expression right) {
        this.left = left;
        this.right = right;
    }
    
    @Override
    public int evaluate() {
        return left.evaluate() * right.evaluate();
    }
    
    @Override
    public String print() {
        return "(" + left.print() + " * " + right.print() + ")";
    }
}

class Subtraction implements Expression {
    private Expression left;
    private Expression right;
    
    public Subtraction(Expression left, Expression right) {
        this.left = left;
        this.right = right;
    }
    
    @Override
    public int evaluate() {
        return left.evaluate() - right.evaluate();
    }
    
    @Override
    public String print() {
        return "(" + left.print() + " - " + right.print() + ")";
    }
}

// Usage
public class Calculator {
    public static void main(String[] args) {
        // Build expression: (5 + 3) * (10 - 2)
        Expression expr = new Multiplication(
            new Addition(new Number(5), new Number(3)),
            new Subtraction(new Number(10), new Number(2))
        );
        
        System.out.println("Expression: " + expr.print());
        System.out.println("Result: " + expr.evaluate());
        // Output:
        // Expression: ((5 + 3) * (10 - 2))
        // Result: 64
        
        // Complex expression: ((2 + 3) * 4) + (10 - 5)
        Expression complex = new Addition(
            new Multiplication(
                new Addition(new Number(2), new Number(3)),
                new Number(4)
            ),
            new Subtraction(new Number(10), new Number(5))
        );
        
        System.out.println("\nExpression: " + complex.print());
        System.out.println("Result: " + complex.evaluate());
        // Expression: (((2 + 3) * 4) + (10 - 5))
        // Result: 25
    }
}
```

---

## When to Use Composite Pattern

### ✅ Use When:
1. You have a **tree/hierarchy** structure
2. You want to treat **individual and group objects** the same way
3. **Part-whole** relationships exist
4. You want **recursive** structure

### ❌ Don't Use When:
1. No hierarchical structure exists
2. Leaf and composite have very different behaviors
3. Simple flat collection is enough

---

## Composite vs Other Patterns

| Pattern | Relationship |
|---------|-------------|
| **Composite** | Tree structure, uniform treatment |
| **Decorator** | Adds behavior, wraps single object |
| **Chain of Responsibility** | Chain, not tree |
| **Iterator** | Often used WITH Composite to traverse |

---

## Two Approaches

### 1. Transparency (Used above)
- Component declares all operations (add, remove)
- Leaves throw exceptions for composite operations
- Simpler client code, less safe

### 2. Safety
- Only Composite has add/remove
- Client must check type before calling add/remove
- Safer, but more complex client code

```java
// Safety approach
interface Component {
    void operation();
}

class Composite implements Component {
    void add(Component c) { }    // Only in Composite
    void remove(Component c) { }
    // ... children management
}
```

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Intent** | Treat individual and composite objects uniformly |
| **Key Idea** | Tree structure where leaves and composites share interface |
| **Benefits** | Simplifies client, easy to add new components |
| **Use Case** | File system, GUIs, org charts, menus |

### Remember:
- **Component** = common interface
- **Leaf** = individual object
- **Composite** = container of components
- **Recursion** = composite operations call child operations

---

**Next: Proxy Pattern →**
