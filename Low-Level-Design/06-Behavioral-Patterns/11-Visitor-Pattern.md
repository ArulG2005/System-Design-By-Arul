# Visitor Design Pattern

## Intent

> **Represent an operation to be performed on the elements of an object structure. Visitor lets you define a new operation without changing the classes of the elements on which it operates.**

---

## The Problem

You have a **structure of different objects** and need to:
- Add **new operations** on them
- Without **modifying** the classes
- **Different operations** for different types

### Bad Approach: Adding methods everywhere

```java
// BAD! Every new operation requires modifying all classes ❌
interface Shape {
    void draw();
    double area();
    void exportXML();      // Added for export feature
    void exportJSON();     // Added for another format
    void calculateCost();  // Added for pricing feature
    // Keep adding methods forever?
}
```

---

## Simple Analogy

Think of a **Tax Auditor** visiting a company:
- Different **departments** exist (Sales, IT, HR)
- Auditor **visits** each department
- Each department **shows its books** differently
- New audit type = new auditor, NOT changing departments

Or think of a **Doctor visiting patients**:
- Different **patient types** (child, adult, elderly)
- Doctor **visits** each
- Same visit, **different examinations** for each type

---

## Structure

```
┌─────────────────────────────────────────────┐
│              «interface» Visitor             │
├─────────────────────────────────────────────┤
│ + visitElementA(ElementA)                    │
│ + visitElementB(ElementB)                    │
│ + visitElementC(ElementC)                    │
└────────────────────────△────────────────────┘
                         │
            ┌────────────┴────────────┐
            │                         │
┌───────────────────┐     ┌───────────────────┐
│  ConcreteVisitor1 │     │  ConcreteVisitor2 │
├───────────────────┤     ├───────────────────┤
│ + visitElementA() │     │ + visitElementA() │
│ + visitElementB() │     │ + visitElementB() │
└───────────────────┘     └───────────────────┘


┌─────────────────────────────────────────────┐
│              «interface» Element             │
├─────────────────────────────────────────────┤
│ + accept(Visitor)                            │
└────────────────────────△────────────────────┘
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│  ElementA   │   │  ElementB   │   │  ElementC   │
├─────────────┤   ├─────────────┤   ├─────────────┤
│ + accept(v) │   │ + accept(v) │   │ + accept(v) │
│ {           │   │ {           │   │ {           │
│  v.visitA() │   │  v.visitB() │   │  v.visitC() │
│ }           │   │ }           │   │ }           │
└─────────────┘   └─────────────┘   └─────────────┘

Key: Double dispatch - Element calls visitor's specific method
```

---

## Basic Example: Document Elements

```java
// Element interface
interface DocumentElement {
    void accept(DocumentVisitor visitor);
}

// Concrete Elements
class TextElement implements DocumentElement {
    private String content;
    
    public TextElement(String content) {
        this.content = content;
    }
    
    public String getContent() { return content; }
    
    @Override
    public void accept(DocumentVisitor visitor) {
        visitor.visitText(this);  // Double dispatch!
    }
}

class ImageElement implements DocumentElement {
    private String filename;
    private int width, height;
    
    public ImageElement(String filename, int width, int height) {
        this.filename = filename;
        this.width = width;
        this.height = height;
    }
    
    public String getFilename() { return filename; }
    public int getWidth() { return width; }
    public int getHeight() { return height; }
    
    @Override
    public void accept(DocumentVisitor visitor) {
        visitor.visitImage(this);
    }
}

class TableElement implements DocumentElement {
    private String[][] data;
    
    public TableElement(String[][] data) {
        this.data = data;
    }
    
    public String[][] getData() { return data; }
    public int getRows() { return data.length; }
    public int getCols() { return data[0].length; }
    
    @Override
    public void accept(DocumentVisitor visitor) {
        visitor.visitTable(this);
    }
}

// Visitor interface
interface DocumentVisitor {
    void visitText(TextElement text);
    void visitImage(ImageElement image);
    void visitTable(TableElement table);
}

// Concrete Visitors

// 1. HTML Export
class HtmlExporter implements DocumentVisitor {
    private StringBuilder html = new StringBuilder();
    
    @Override
    public void visitText(TextElement text) {
        html.append("<p>").append(text.getContent()).append("</p>\n");
    }
    
    @Override
    public void visitImage(ImageElement image) {
        html.append(String.format("<img src=\"%s\" width=\"%d\" height=\"%d\"/>\n",
            image.getFilename(), image.getWidth(), image.getHeight()));
    }
    
    @Override
    public void visitTable(TableElement table) {
        html.append("<table>\n");
        for (String[] row : table.getData()) {
            html.append("  <tr>\n");
            for (String cell : row) {
                html.append("    <td>").append(cell).append("</td>\n");
            }
            html.append("  </tr>\n");
        }
        html.append("</table>\n");
    }
    
    public String getHtml() { return html.toString(); }
}

// 2. Plain Text Export
class PlainTextExporter implements DocumentVisitor {
    private StringBuilder text = new StringBuilder();
    
    @Override
    public void visitText(TextElement textEl) {
        text.append(textEl.getContent()).append("\n\n");
    }
    
    @Override
    public void visitImage(ImageElement image) {
        text.append("[Image: ").append(image.getFilename()).append("]\n\n");
    }
    
    @Override
    public void visitTable(TableElement table) {
        for (String[] row : table.getData()) {
            text.append(String.join(" | ", row)).append("\n");
        }
        text.append("\n");
    }
    
    public String getText() { return text.toString(); }
}

// 3. Word Count
class WordCounter implements DocumentVisitor {
    private int wordCount = 0;
    private int imageCount = 0;
    private int tableCount = 0;
    
    @Override
    public void visitText(TextElement text) {
        wordCount += text.getContent().split("\\s+").length;
    }
    
    @Override
    public void visitImage(ImageElement image) {
        imageCount++;
    }
    
    @Override
    public void visitTable(TableElement table) {
        tableCount++;
        // Count words in table cells
        for (String[] row : table.getData()) {
            for (String cell : row) {
                wordCount += cell.split("\\s+").length;
            }
        }
    }
    
    public void printStats() {
        System.out.println("Document Statistics:");
        System.out.println("  Words: " + wordCount);
        System.out.println("  Images: " + imageCount);
        System.out.println("  Tables: " + tableCount);
    }
}

// Document structure
class Document {
    private List<DocumentElement> elements = new ArrayList<>();
    
    public void add(DocumentElement element) {
        elements.add(element);
    }
    
    public void accept(DocumentVisitor visitor) {
        for (DocumentElement element : elements) {
            element.accept(visitor);
        }
    }
}

// Usage
public class DocumentDemo {
    public static void main(String[] args) {
        // Create document
        Document doc = new Document();
        doc.add(new TextElement("Welcome to our report"));
        doc.add(new ImageElement("chart.png", 800, 600));
        doc.add(new TextElement("Here are the quarterly results:"));
        doc.add(new TableElement(new String[][] {
            {"Q1", "Q2", "Q3", "Q4"},
            {"$100K", "$120K", "$150K", "$180K"}
        }));
        doc.add(new TextElement("Thank you for reading!"));
        
        // Export to HTML
        HtmlExporter htmlExporter = new HtmlExporter();
        doc.accept(htmlExporter);
        System.out.println("=== HTML OUTPUT ===");
        System.out.println(htmlExporter.getHtml());
        
        // Export to plain text
        PlainTextExporter textExporter = new PlainTextExporter();
        doc.accept(textExporter);
        System.out.println("=== PLAIN TEXT ===");
        System.out.println(textExporter.getText());
        
        // Count statistics
        WordCounter counter = new WordCounter();
        doc.accept(counter);
        counter.printStats();
    }
}
```

---

## Real-World Examples

### Example 1: Shopping Cart with Different Operations

```java
// Element interface
interface ShoppingItem {
    void accept(ShoppingVisitor visitor);
}

// Concrete items
class Book implements ShoppingItem {
    private String title;
    private double price;
    private String isbn;
    
    public Book(String title, double price, String isbn) {
        this.title = title;
        this.price = price;
        this.isbn = isbn;
    }
    
    public String getTitle() { return title; }
    public double getPrice() { return price; }
    public String getIsbn() { return isbn; }
    
    @Override
    public void accept(ShoppingVisitor visitor) {
        visitor.visitBook(this);
    }
}

class Electronics implements ShoppingItem {
    private String name;
    private double price;
    private String model;
    private double weight;  // for shipping calculation
    
    public Electronics(String name, double price, String model, double weight) {
        this.name = name;
        this.price = price;
        this.model = model;
        this.weight = weight;
    }
    
    public String getName() { return name; }
    public double getPrice() { return price; }
    public String getModel() { return model; }
    public double getWeight() { return weight; }
    
    @Override
    public void accept(ShoppingVisitor visitor) {
        visitor.visitElectronics(this);
    }
}

class Grocery implements ShoppingItem {
    private String name;
    private double price;
    private double weight;
    private boolean perishable;
    
    public Grocery(String name, double price, double weight, boolean perishable) {
        this.name = name;
        this.price = price;
        this.weight = weight;
        this.perishable = perishable;
    }
    
    public String getName() { return name; }
    public double getPrice() { return price; }
    public double getWeight() { return weight; }
    public boolean isPerishable() { return perishable; }
    
    @Override
    public void accept(ShoppingVisitor visitor) {
        visitor.visitGrocery(this);
    }
}

// Visitor interface
interface ShoppingVisitor {
    void visitBook(Book book);
    void visitElectronics(Electronics electronics);
    void visitGrocery(Grocery grocery);
}

// Price Calculator (includes taxes)
class PriceCalculator implements ShoppingVisitor {
    private double total = 0;
    
    @Override
    public void visitBook(Book book) {
        // Books: no tax
        total += book.getPrice();
        System.out.println("Book: $" + book.getPrice() + " (no tax)");
    }
    
    @Override
    public void visitElectronics(Electronics electronics) {
        // Electronics: 15% tax
        double tax = electronics.getPrice() * 0.15;
        double itemTotal = electronics.getPrice() + tax;
        total += itemTotal;
        System.out.println("Electronics: $" + electronics.getPrice() 
            + " + $" + tax + " tax = $" + itemTotal);
    }
    
    @Override
    public void visitGrocery(Grocery grocery) {
        // Grocery: 5% tax
        double tax = grocery.getPrice() * 0.05;
        double itemTotal = grocery.getPrice() + tax;
        total += itemTotal;
        System.out.println("Grocery: $" + grocery.getPrice() 
            + " + $" + tax + " tax = $" + itemTotal);
    }
    
    public double getTotal() { return total; }
}

// Shipping Calculator
class ShippingCalculator implements ShoppingVisitor {
    private double shippingCost = 0;
    private int deliveryDays = 3;  // Standard
    
    @Override
    public void visitBook(Book book) {
        // Books: flat rate
        shippingCost += 2.99;
    }
    
    @Override
    public void visitElectronics(Electronics electronics) {
        // Electronics: weight-based + insurance
        shippingCost += electronics.getWeight() * 2.0 + 5.0;  // $2/lb + $5 insurance
    }
    
    @Override
    public void visitGrocery(Grocery grocery) {
        // Grocery: expedited if perishable
        if (grocery.isPerishable()) {
            shippingCost += 7.99;  // Express shipping
            deliveryDays = 1;
        } else {
            shippingCost += grocery.getWeight() * 0.50;
        }
    }
    
    public double getShippingCost() { return shippingCost; }
    public int getDeliveryDays() { return deliveryDays; }
}

// Inventory Report
class InventoryReporter implements ShoppingVisitor {
    private List<String> report = new ArrayList<>();
    
    @Override
    public void visitBook(Book book) {
        report.add(String.format("BOOK: %s (ISBN: %s) - $%.2f", 
            book.getTitle(), book.getIsbn(), book.getPrice()));
    }
    
    @Override
    public void visitElectronics(Electronics electronics) {
        report.add(String.format("ELECTRONICS: %s (Model: %s) - $%.2f, %.1f lbs", 
            electronics.getName(), electronics.getModel(), 
            electronics.getPrice(), electronics.getWeight()));
    }
    
    @Override
    public void visitGrocery(Grocery grocery) {
        String perishLabel = grocery.isPerishable() ? " [PERISHABLE]" : "";
        report.add(String.format("GROCERY: %s - $%.2f, %.1f lbs%s", 
            grocery.getName(), grocery.getPrice(), 
            grocery.getWeight(), perishLabel));
    }
    
    public void printReport() {
        System.out.println("=== INVENTORY REPORT ===");
        report.forEach(System.out::println);
    }
}

// Shopping Cart
class ShoppingCart {
    private List<ShoppingItem> items = new ArrayList<>();
    
    public void add(ShoppingItem item) {
        items.add(item);
    }
    
    public void accept(ShoppingVisitor visitor) {
        for (ShoppingItem item : items) {
            item.accept(visitor);
        }
    }
}

// Usage
public class ShoppingDemo {
    public static void main(String[] args) {
        ShoppingCart cart = new ShoppingCart();
        cart.add(new Book("Design Patterns", 49.99, "978-0201633610"));
        cart.add(new Electronics("Laptop", 999.99, "XPS-15", 4.5));
        cart.add(new Grocery("Milk", 3.99, 1.0, true));
        cart.add(new Grocery("Rice", 8.99, 5.0, false));
        cart.add(new Book("Clean Code", 39.99, "978-0132350884"));
        
        // Calculate prices
        System.out.println("=== PRICE CALCULATION ===");
        PriceCalculator priceCalc = new PriceCalculator();
        cart.accept(priceCalc);
        System.out.println("TOTAL: $" + String.format("%.2f", priceCalc.getTotal()));
        
        // Calculate shipping
        System.out.println("\n=== SHIPPING CALCULATION ===");
        ShippingCalculator shipCalc = new ShippingCalculator();
        cart.accept(shipCalc);
        System.out.println("Shipping: $" + String.format("%.2f", shipCalc.getShippingCost()));
        System.out.println("Delivery: " + shipCalc.getDeliveryDays() + " days");
        
        // Generate inventory report
        System.out.println();
        InventoryReporter reporter = new InventoryReporter();
        cart.accept(reporter);
        reporter.printReport();
    }
}
```

---

### Example 2: AST (Abstract Syntax Tree) Operations

```java
// AST Node interface
interface AstNode {
    void accept(AstVisitor visitor);
}

// Expression nodes
class NumberNode implements AstNode {
    private double value;
    
    public NumberNode(double value) {
        this.value = value;
    }
    
    public double getValue() { return value; }
    
    @Override
    public void accept(AstVisitor visitor) {
        visitor.visitNumber(this);
    }
}

class BinaryOpNode implements AstNode {
    private AstNode left;
    private AstNode right;
    private String operator;
    
    public BinaryOpNode(AstNode left, String operator, AstNode right) {
        this.left = left;
        this.operator = operator;
        this.right = right;
    }
    
    public AstNode getLeft() { return left; }
    public AstNode getRight() { return right; }
    public String getOperator() { return operator; }
    
    @Override
    public void accept(AstVisitor visitor) {
        visitor.visitBinaryOp(this);
    }
}

class VariableNode implements AstNode {
    private String name;
    
    public VariableNode(String name) {
        this.name = name;
    }
    
    public String getName() { return name; }
    
    @Override
    public void accept(AstVisitor visitor) {
        visitor.visitVariable(this);
    }
}

class FunctionCallNode implements AstNode {
    private String functionName;
    private List<AstNode> arguments;
    
    public FunctionCallNode(String name, List<AstNode> args) {
        this.functionName = name;
        this.arguments = args;
    }
    
    public String getFunctionName() { return functionName; }
    public List<AstNode> getArguments() { return arguments; }
    
    @Override
    public void accept(AstVisitor visitor) {
        visitor.visitFunctionCall(this);
    }
}

// Visitor interface
interface AstVisitor {
    void visitNumber(NumberNode node);
    void visitBinaryOp(BinaryOpNode node);
    void visitVariable(VariableNode node);
    void visitFunctionCall(FunctionCallNode node);
}

// Print Visitor - creates readable expression
class PrintVisitor implements AstVisitor {
    private StringBuilder output = new StringBuilder();
    
    @Override
    public void visitNumber(NumberNode node) {
        output.append(node.getValue());
    }
    
    @Override
    public void visitBinaryOp(BinaryOpNode node) {
        output.append("(");
        node.getLeft().accept(this);
        output.append(" ").append(node.getOperator()).append(" ");
        node.getRight().accept(this);
        output.append(")");
    }
    
    @Override
    public void visitVariable(VariableNode node) {
        output.append(node.getName());
    }
    
    @Override
    public void visitFunctionCall(FunctionCallNode node) {
        output.append(node.getFunctionName()).append("(");
        List<AstNode> args = node.getArguments();
        for (int i = 0; i < args.size(); i++) {
            args.get(i).accept(this);
            if (i < args.size() - 1) output.append(", ");
        }
        output.append(")");
    }
    
    public String getOutput() { return output.toString(); }
}

// Evaluate Visitor - calculates result
class EvaluateVisitor implements AstVisitor {
    private Map<String, Double> variables = new HashMap<>();
    private Stack<Double> results = new Stack<>();
    
    public void setVariable(String name, double value) {
        variables.put(name, value);
    }
    
    @Override
    public void visitNumber(NumberNode node) {
        results.push(node.getValue());
    }
    
    @Override
    public void visitBinaryOp(BinaryOpNode node) {
        node.getLeft().accept(this);
        node.getRight().accept(this);
        
        double right = results.pop();
        double left = results.pop();
        
        double result = switch (node.getOperator()) {
            case "+" -> left + right;
            case "-" -> left - right;
            case "*" -> left * right;
            case "/" -> left / right;
            case "^" -> Math.pow(left, right);
            default -> throw new RuntimeException("Unknown operator: " + node.getOperator());
        };
        
        results.push(result);
    }
    
    @Override
    public void visitVariable(VariableNode node) {
        Double value = variables.get(node.getName());
        if (value == null) {
            throw new RuntimeException("Undefined variable: " + node.getName());
        }
        results.push(value);
    }
    
    @Override
    public void visitFunctionCall(FunctionCallNode node) {
        // Evaluate all arguments
        List<Double> argValues = new ArrayList<>();
        for (AstNode arg : node.getArguments()) {
            arg.accept(this);
            argValues.add(results.pop());
        }
        
        double result = switch (node.getFunctionName()) {
            case "sin" -> Math.sin(argValues.get(0));
            case "cos" -> Math.cos(argValues.get(0));
            case "sqrt" -> Math.sqrt(argValues.get(0));
            case "max" -> Math.max(argValues.get(0), argValues.get(1));
            case "min" -> Math.min(argValues.get(0), argValues.get(1));
            default -> throw new RuntimeException("Unknown function: " + node.getFunctionName());
        };
        
        results.push(result);
    }
    
    public double getResult() {
        return results.isEmpty() ? 0 : results.peek();
    }
}

// Usage
public class AstDemo {
    public static void main(String[] args) {
        // Build AST for: (x + 5) * sqrt(y)
        AstNode expression = new BinaryOpNode(
            new BinaryOpNode(
                new VariableNode("x"),
                "+",
                new NumberNode(5)
            ),
            "*",
            new FunctionCallNode("sqrt", Arrays.asList(new VariableNode("y")))
        );
        
        // Print expression
        PrintVisitor printer = new PrintVisitor();
        expression.accept(printer);
        System.out.println("Expression: " + printer.getOutput());
        
        // Evaluate with values
        EvaluateVisitor evaluator = new EvaluateVisitor();
        evaluator.setVariable("x", 3);  // x = 3
        evaluator.setVariable("y", 16); // y = 16
        expression.accept(evaluator);
        System.out.println("Result (x=3, y=16): " + evaluator.getResult());
        // (3 + 5) * sqrt(16) = 8 * 4 = 32
    }
}
```

---

### Example 3: File System Operations

```java
// File System Element
interface FileSystemElement {
    String getName();
    void accept(FileSystemVisitor visitor);
}

// File
class File implements FileSystemElement {
    private String name;
    private long size;
    private String extension;
    
    public File(String name, long size) {
        this.name = name;
        this.size = size;
        int dotIndex = name.lastIndexOf('.');
        this.extension = dotIndex > 0 ? name.substring(dotIndex + 1) : "";
    }
    
    @Override
    public String getName() { return name; }
    public long getSize() { return size; }
    public String getExtension() { return extension; }
    
    @Override
    public void accept(FileSystemVisitor visitor) {
        visitor.visitFile(this);
    }
}

// Directory
class Directory implements FileSystemElement {
    private String name;
    private List<FileSystemElement> children = new ArrayList<>();
    
    public Directory(String name) {
        this.name = name;
    }
    
    public void add(FileSystemElement element) {
        children.add(element);
    }
    
    @Override
    public String getName() { return name; }
    public List<FileSystemElement> getChildren() { return children; }
    
    @Override
    public void accept(FileSystemVisitor visitor) {
        visitor.visitDirectory(this);
    }
}

// Visitor interface
interface FileSystemVisitor {
    void visitFile(File file);
    void visitDirectory(Directory directory);
}

// Size Calculator
class SizeCalculator implements FileSystemVisitor {
    private long totalSize = 0;
    
    @Override
    public void visitFile(File file) {
        totalSize += file.getSize();
    }
    
    @Override
    public void visitDirectory(Directory directory) {
        for (FileSystemElement child : directory.getChildren()) {
            child.accept(this);
        }
    }
    
    public long getTotalSize() { return totalSize; }
    
    public String getHumanReadableSize() {
        if (totalSize < 1024) return totalSize + " B";
        if (totalSize < 1024 * 1024) return (totalSize / 1024) + " KB";
        return (totalSize / (1024 * 1024)) + " MB";
    }
}

// File Type Counter
class FileTypeCounter implements FileSystemVisitor {
    private Map<String, Integer> typeCounts = new HashMap<>();
    private Map<String, Long> typeSizes = new HashMap<>();
    
    @Override
    public void visitFile(File file) {
        String ext = file.getExtension().isEmpty() ? "no extension" : file.getExtension();
        typeCounts.merge(ext, 1, Integer::sum);
        typeSizes.merge(ext, file.getSize(), Long::sum);
    }
    
    @Override
    public void visitDirectory(Directory directory) {
        for (FileSystemElement child : directory.getChildren()) {
            child.accept(this);
        }
    }
    
    public void printReport() {
        System.out.println("=== FILE TYPE REPORT ===");
        typeCounts.forEach((type, count) -> {
            long size = typeSizes.get(type);
            System.out.printf("%s: %d files, %d bytes\n", type, count, size);
        });
    }
}

// Search Visitor
class SearchVisitor implements FileSystemVisitor {
    private String searchPattern;
    private List<String> results = new ArrayList<>();
    private String currentPath = "";
    
    public SearchVisitor(String pattern) {
        this.searchPattern = pattern.toLowerCase();
    }
    
    @Override
    public void visitFile(File file) {
        if (file.getName().toLowerCase().contains(searchPattern)) {
            results.add(currentPath + "/" + file.getName());
        }
    }
    
    @Override
    public void visitDirectory(Directory directory) {
        String previousPath = currentPath;
        currentPath = currentPath + "/" + directory.getName();
        
        if (directory.getName().toLowerCase().contains(searchPattern)) {
            results.add(currentPath);
        }
        
        for (FileSystemElement child : directory.getChildren()) {
            child.accept(this);
        }
        
        currentPath = previousPath;
    }
    
    public List<String> getResults() { return results; }
}

// Tree Printer
class TreePrinter implements FileSystemVisitor {
    private int depth = 0;
    
    @Override
    public void visitFile(File file) {
        System.out.println(indent() + "📄 " + file.getName() + " (" + file.getSize() + " bytes)");
    }
    
    @Override
    public void visitDirectory(Directory directory) {
        System.out.println(indent() + "📁 " + directory.getName() + "/");
        depth++;
        for (FileSystemElement child : directory.getChildren()) {
            child.accept(this);
        }
        depth--;
    }
    
    private String indent() {
        return "  ".repeat(depth);
    }
}

// Usage
public class FileSystemDemo {
    public static void main(String[] args) {
        // Build file system
        Directory root = new Directory("project");
        
        Directory src = new Directory("src");
        src.add(new File("Main.java", 2500));
        src.add(new File("Utils.java", 1200));
        src.add(new File("Config.java", 800));
        
        Directory tests = new Directory("tests");
        tests.add(new File("MainTest.java", 1500));
        tests.add(new File("UtilsTest.java", 900));
        
        Directory docs = new Directory("docs");
        docs.add(new File("README.md", 3000));
        docs.add(new File("API.md", 5000));
        docs.add(new File("architecture.png", 150000));
        
        root.add(src);
        root.add(tests);
        root.add(docs);
        root.add(new File("pom.xml", 2000));
        root.add(new File(".gitignore", 100));
        
        // Print tree
        System.out.println("=== FILE TREE ===");
        TreePrinter printer = new TreePrinter();
        root.accept(printer);
        
        // Calculate size
        System.out.println("\n=== SIZE CALCULATION ===");
        SizeCalculator sizeCalc = new SizeCalculator();
        root.accept(sizeCalc);
        System.out.println("Total size: " + sizeCalc.getHumanReadableSize());
        
        // File type report
        System.out.println();
        FileTypeCounter counter = new FileTypeCounter();
        root.accept(counter);
        counter.printReport();
        
        // Search
        System.out.println("\n=== SEARCH: 'test' ===");
        SearchVisitor searcher = new SearchVisitor("test");
        root.accept(searcher);
        searcher.getResults().forEach(System.out::println);
    }
}
```

---

## Double Dispatch Explained

```java
// Single Dispatch (normal): Method chosen by receiver type only
element.process();  // Calls element's type's method

// Double Dispatch (Visitor): Method chosen by BOTH types!
element.accept(visitor);          // 1. Dispatch on element type
// Inside accept:
visitor.visitConcreteType(this);  // 2. Dispatch on visitor type

// Result: Correct method for BOTH element AND visitor type!
```

---

## When to Use

### ✅ Use When:
1. **Many operations** on a structure of objects
2. Classes are **stable** but operations change often
3. Need to perform operations on **unrelated classes**
4. Want to **avoid polluting** classes with operations

### ❌ Don't Use When:
1. Class hierarchy changes **frequently**
2. Only **one or two** operations needed
3. Adding visitor = modifying **all element** classes

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Intent** | Add operations without modifying classes |
| **Key Idea** | Double dispatch - element.accept(visitor) |
| **Benefits** | SRP, OCP for new operations, gather related operations |
| **Drawback** | Adding new element = updating all visitors |
| **Use Cases** | Compilers, exporters, validators, calculators |

### Remember:
- **Element** → `accept(visitor)` { `visitor.visitMe(this)` }
- **Visitor** → Has a method for each element type
- Easy to add **new operations** (new visitor)
- Hard to add **new elements** (all visitors need update)

---

This completes the **Behavioral Patterns** section!

**Next: Multithreading and Concurrency →**
