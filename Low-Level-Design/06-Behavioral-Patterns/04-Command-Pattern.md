# Command Design Pattern

## Intent

> **Encapsulate a request as an object, thereby letting you parameterize clients with different requests, queue or log requests, and support undoable operations.**

---

## The Problem

You need to:
- **Decouple** the object that invokes an operation from the one that performs it
- **Queue** requests for later execution
- **Log** operations for audit or recovery
- **Undo/Redo** operations

---

## Simple Analogy

Think of a **Restaurant**:
- You (Client) don't talk directly to the Chef (Receiver)
- You write your order on a slip (Command)
- Waiter takes the slip (Invoker)
- Chef reads and executes the order
- Order slips can be queued, logged, even cancelled!

Or think of a **TV Remote**:
- Remote button = Command object
- Remote = Invoker
- TV = Receiver
- Each button encapsulates a different command

---

## Structure

```
┌──────────────────┐         ┌──────────────────────────┐
│     Client       │         │        Invoker           │
└────────┬─────────┘         ├──────────────────────────┤
         │ creates           │ - command: Command       │
         ▼                   │ + setCommand(Command)    │
┌──────────────────────────┐ │ + executeCommand()       │
│     «interface»          │ └────────────┬─────────────┘
│       Command            │◄─────────────┘
├──────────────────────────┤
│ + execute()              │
│ + undo()                 │
└────────────△─────────────┘
             │
    ┌────────┴────────┐
    │                 │
┌──────────────┐ ┌──────────────┐
│ CommandA     │ │ CommandB     │
├──────────────┤ ├──────────────┤        ┌─────────────┐
│ - receiver   │ │ - receiver   │───────>│  Receiver   │
│ + execute()  │ │ + execute()  │        ├─────────────┤
│ + undo()     │ │ + undo()     │        │ + action()  │
└──────────────┘ └──────────────┘        └─────────────┘
```

---

## Basic Example: Remote Control

```java
// Command interface
interface Command {
    void execute();
    void undo();
}

// Receiver
class Light {
    private String location;
    
    public Light(String location) {
        this.location = location;
    }
    
    public void on() {
        System.out.println(location + " light is ON");
    }
    
    public void off() {
        System.out.println(location + " light is OFF");
    }
}

// Concrete Commands
class LightOnCommand implements Command {
    private Light light;
    
    public LightOnCommand(Light light) {
        this.light = light;
    }
    
    @Override
    public void execute() {
        light.on();
    }
    
    @Override
    public void undo() {
        light.off();
    }
}

class LightOffCommand implements Command {
    private Light light;
    
    public LightOffCommand(Light light) {
        this.light = light;
    }
    
    @Override
    public void execute() {
        light.off();
    }
    
    @Override
    public void undo() {
        light.on();
    }
}

// More receivers
class Fan {
    private int speed = 0; // 0=off, 1=low, 2=medium, 3=high
    
    public void low() { speed = 1; System.out.println("Fan is on LOW"); }
    public void medium() { speed = 2; System.out.println("Fan is on MEDIUM"); }
    public void high() { speed = 3; System.out.println("Fan is on HIGH"); }
    public void off() { speed = 0; System.out.println("Fan is OFF"); }
    public int getSpeed() { return speed; }
}

class FanHighCommand implements Command {
    private Fan fan;
    private int prevSpeed;
    
    public FanHighCommand(Fan fan) {
        this.fan = fan;
    }
    
    @Override
    public void execute() {
        prevSpeed = fan.getSpeed();
        fan.high();
    }
    
    @Override
    public void undo() {
        switch (prevSpeed) {
            case 0: fan.off(); break;
            case 1: fan.low(); break;
            case 2: fan.medium(); break;
            case 3: fan.high(); break;
        }
    }
}

// Null Object pattern for empty slots
class NoCommand implements Command {
    @Override
    public void execute() { }
    @Override
    public void undo() { }
}

// Invoker
class RemoteControl {
    private Command[] onCommands;
    private Command[] offCommands;
    private Command lastCommand;
    
    public RemoteControl(int slots) {
        onCommands = new Command[slots];
        offCommands = new Command[slots];
        
        // Initialize with NoCommand
        Command noCommand = new NoCommand();
        for (int i = 0; i < slots; i++) {
            onCommands[i] = noCommand;
            offCommands[i] = noCommand;
        }
        lastCommand = noCommand;
    }
    
    public void setCommand(int slot, Command onCommand, Command offCommand) {
        onCommands[slot] = onCommand;
        offCommands[slot] = offCommand;
    }
    
    public void onButtonPressed(int slot) {
        onCommands[slot].execute();
        lastCommand = onCommands[slot];
    }
    
    public void offButtonPressed(int slot) {
        offCommands[slot].execute();
        lastCommand = offCommands[slot];
    }
    
    public void undoButtonPressed() {
        System.out.println("--- UNDO ---");
        lastCommand.undo();
    }
}

// Usage
public class HomeAutomation {
    public static void main(String[] args) {
        RemoteControl remote = new RemoteControl(3);
        
        // Setup devices
        Light livingRoomLight = new Light("Living Room");
        Light kitchenLight = new Light("Kitchen");
        Fan fan = new Fan();
        
        // Configure remote
        remote.setCommand(0, 
            new LightOnCommand(livingRoomLight),
            new LightOffCommand(livingRoomLight));
        remote.setCommand(1,
            new LightOnCommand(kitchenLight),
            new LightOffCommand(kitchenLight));
        remote.setCommand(2,
            new FanHighCommand(fan),
            new FanOffCommand(fan));
        
        // Use remote
        remote.onButtonPressed(0);   // Living Room light ON
        remote.offButtonPressed(0);  // Living Room light OFF
        remote.undoButtonPressed();  // Living Room light ON (undo)
        
        remote.onButtonPressed(2);   // Fan HIGH
        remote.undoButtonPressed();  // Fan OFF (undo)
    }
}
```

---

## Real-World Examples

### Example 1: Text Editor with Undo/Redo

```java
// Command interface
interface TextCommand {
    void execute();
    void undo();
    String getDescription();
}

// Receiver
class TextDocument {
    private StringBuilder content = new StringBuilder();
    private int cursorPosition = 0;
    
    public void insert(int position, String text) {
        content.insert(position, text);
        cursorPosition = position + text.length();
    }
    
    public void delete(int start, int length) {
        content.delete(start, start + length);
        cursorPosition = start;
    }
    
    public String getContent() {
        return content.toString();
    }
    
    public int getCursorPosition() {
        return cursorPosition;
    }
    
    public void setCursorPosition(int pos) {
        cursorPosition = pos;
    }
}

// Concrete Commands
class InsertCommand implements TextCommand {
    private TextDocument document;
    private int position;
    private String text;
    
    public InsertCommand(TextDocument document, int position, String text) {
        this.document = document;
        this.position = position;
        this.text = text;
    }
    
    @Override
    public void execute() {
        document.insert(position, text);
    }
    
    @Override
    public void undo() {
        document.delete(position, text.length());
    }
    
    @Override
    public String getDescription() {
        return "Insert '" + text + "' at position " + position;
    }
}

class DeleteCommand implements TextCommand {
    private TextDocument document;
    private int position;
    private int length;
    private String deletedText;  // Store for undo
    
    public DeleteCommand(TextDocument document, int position, int length) {
        this.document = document;
        this.position = position;
        this.length = length;
    }
    
    @Override
    public void execute() {
        // Store deleted text for undo
        deletedText = document.getContent().substring(position, position + length);
        document.delete(position, length);
    }
    
    @Override
    public void undo() {
        document.insert(position, deletedText);
    }
    
    @Override
    public String getDescription() {
        return "Delete " + length + " chars at position " + position;
    }
}

// Invoker with history
class TextEditor {
    private TextDocument document;
    private Deque<TextCommand> undoStack = new ArrayDeque<>();
    private Deque<TextCommand> redoStack = new ArrayDeque<>();
    
    public TextEditor() {
        this.document = new TextDocument();
    }
    
    public void executeCommand(TextCommand command) {
        command.execute();
        undoStack.push(command);
        redoStack.clear();  // Clear redo when new command executed
        System.out.println("Executed: " + command.getDescription());
    }
    
    public void undo() {
        if (!undoStack.isEmpty()) {
            TextCommand command = undoStack.pop();
            command.undo();
            redoStack.push(command);
            System.out.println("Undone: " + command.getDescription());
        } else {
            System.out.println("Nothing to undo!");
        }
    }
    
    public void redo() {
        if (!redoStack.isEmpty()) {
            TextCommand command = redoStack.pop();
            command.execute();
            undoStack.push(command);
            System.out.println("Redone: " + command.getDescription());
        } else {
            System.out.println("Nothing to redo!");
        }
    }
    
    public void type(String text) {
        int pos = document.getCursorPosition();
        executeCommand(new InsertCommand(document, pos, text));
    }
    
    public void backspace(int count) {
        int pos = document.getCursorPosition();
        if (pos >= count) {
            executeCommand(new DeleteCommand(document, pos - count, count));
        }
    }
    
    public void showContent() {
        System.out.println("Document: \"" + document.getContent() + "\"");
    }
}

// Usage
public class TextEditorApp {
    public static void main(String[] args) {
        TextEditor editor = new TextEditor();
        
        editor.type("Hello");
        editor.showContent();  // "Hello"
        
        editor.type(" World");
        editor.showContent();  // "Hello World"
        
        editor.type("!");
        editor.showContent();  // "Hello World!"
        
        System.out.println("\n--- Undo Operations ---");
        
        editor.undo();
        editor.showContent();  // "Hello World"
        
        editor.undo();
        editor.showContent();  // "Hello"
        
        System.out.println("\n--- Redo Operations ---");
        
        editor.redo();
        editor.showContent();  // "Hello World"
        
        editor.type(" Java");
        editor.showContent();  // "Hello World Java"
        
        editor.redo();  // Nothing to redo (cleared after new command)
    }
}
```

---

### Example 2: Order Processing System

```java
// Command interface
interface OrderCommand {
    void execute();
    void undo();
}

// Receiver
class Order {
    private String orderId;
    private String status;
    private List<String> history = new ArrayList<>();
    
    public Order(String orderId) {
        this.orderId = orderId;
        this.status = "CREATED";
        log("Order created");
    }
    
    public void process() {
        status = "PROCESSING";
        log("Order is being processed");
    }
    
    public void ship() {
        status = "SHIPPED";
        log("Order shipped");
    }
    
    public void deliver() {
        status = "DELIVERED";
        log("Order delivered");
    }
    
    public void cancel() {
        status = "CANCELLED";
        log("Order cancelled");
    }
    
    public void setStatus(String status) {
        this.status = status;
        log("Status changed to " + status);
    }
    
    private void log(String message) {
        history.add(message);
        System.out.println("[" + orderId + "] " + message);
    }
    
    public String getStatus() { return status; }
    public String getOrderId() { return orderId; }
}

// Concrete Commands
class ProcessOrderCommand implements OrderCommand {
    private Order order;
    private String previousStatus;
    
    public ProcessOrderCommand(Order order) {
        this.order = order;
    }
    
    @Override
    public void execute() {
        previousStatus = order.getStatus();
        order.process();
    }
    
    @Override
    public void undo() {
        order.setStatus(previousStatus);
    }
}

class ShipOrderCommand implements OrderCommand {
    private Order order;
    private String previousStatus;
    
    public ShipOrderCommand(Order order) {
        this.order = order;
    }
    
    @Override
    public void execute() {
        previousStatus = order.getStatus();
        order.ship();
    }
    
    @Override
    public void undo() {
        order.setStatus(previousStatus);
    }
}

class CancelOrderCommand implements OrderCommand {
    private Order order;
    private String previousStatus;
    
    public CancelOrderCommand(Order order) {
        this.order = order;
    }
    
    @Override
    public void execute() {
        previousStatus = order.getStatus();
        order.cancel();
    }
    
    @Override
    public void undo() {
        order.setStatus(previousStatus);
    }
}

// Macro Command - execute multiple commands
class MacroCommand implements OrderCommand {
    private List<OrderCommand> commands;
    
    public MacroCommand(List<OrderCommand> commands) {
        this.commands = new ArrayList<>(commands);
    }
    
    @Override
    public void execute() {
        for (OrderCommand command : commands) {
            command.execute();
        }
    }
    
    @Override
    public void undo() {
        // Undo in reverse order
        for (int i = commands.size() - 1; i >= 0; i--) {
            commands.get(i).undo();
        }
    }
}

// Invoker with queue
class OrderProcessor {
    private Queue<OrderCommand> commandQueue = new LinkedList<>();
    private List<OrderCommand> executedCommands = new ArrayList<>();
    
    public void addToQueue(OrderCommand command) {
        commandQueue.add(command);
        System.out.println("Command added to queue");
    }
    
    public void processQueue() {
        System.out.println("\n=== Processing Command Queue ===");
        while (!commandQueue.isEmpty()) {
            OrderCommand command = commandQueue.poll();
            command.execute();
            executedCommands.add(command);
        }
    }
    
    public void undoLast() {
        if (!executedCommands.isEmpty()) {
            OrderCommand command = executedCommands.remove(executedCommands.size() - 1);
            command.undo();
        }
    }
}

// Usage
public class OrderSystem {
    public static void main(String[] args) {
        OrderProcessor processor = new OrderProcessor();
        
        Order order1 = new Order("ORD-001");
        Order order2 = new Order("ORD-002");
        
        // Queue up commands
        processor.addToQueue(new ProcessOrderCommand(order1));
        processor.addToQueue(new ProcessOrderCommand(order2));
        processor.addToQueue(new ShipOrderCommand(order1));
        
        // Process all at once
        processor.processQueue();
        
        System.out.println("\n=== Undo last operation ===");
        processor.undoLast();  // Undo ship order1
        
        System.out.println("\nOrder 1 status: " + order1.getStatus());
        System.out.println("Order 2 status: " + order2.getStatus());
    }
}
```

---

### Example 3: Database Transaction

```java
// Command interface
interface DBCommand {
    void execute() throws Exception;
    void undo() throws Exception;
}

// Simple mock database
class Database {
    private Map<String, Object> data = new HashMap<>();
    
    public void insert(String table, String key, Object value) {
        String fullKey = table + ":" + key;
        data.put(fullKey, value);
        System.out.println("INSERT into " + table + ": " + key + " = " + value);
    }
    
    public void update(String table, String key, Object value) {
        String fullKey = table + ":" + key;
        data.put(fullKey, value);
        System.out.println("UPDATE " + table + ": " + key + " = " + value);
    }
    
    public void delete(String table, String key) {
        String fullKey = table + ":" + key;
        data.remove(fullKey);
        System.out.println("DELETE from " + table + ": " + key);
    }
    
    public Object get(String table, String key) {
        return data.get(table + ":" + key);
    }
}

// Concrete Commands
class InsertDBCommand implements DBCommand {
    private Database db;
    private String table;
    private String key;
    private Object value;
    
    public InsertDBCommand(Database db, String table, String key, Object value) {
        this.db = db;
        this.table = table;
        this.key = key;
        this.value = value;
    }
    
    @Override
    public void execute() {
        db.insert(table, key, value);
    }
    
    @Override
    public void undo() {
        db.delete(table, key);
    }
}

class UpdateDBCommand implements DBCommand {
    private Database db;
    private String table;
    private String key;
    private Object newValue;
    private Object oldValue;
    
    public UpdateDBCommand(Database db, String table, String key, Object newValue) {
        this.db = db;
        this.table = table;
        this.key = key;
        this.newValue = newValue;
    }
    
    @Override
    public void execute() {
        oldValue = db.get(table, key);  // Store old value
        db.update(table, key, newValue);
    }
    
    @Override
    public void undo() {
        if (oldValue != null) {
            db.update(table, key, oldValue);
        } else {
            db.delete(table, key);
        }
    }
}

// Transaction - groups commands
class Transaction {
    private List<DBCommand> commands = new ArrayList<>();
    private List<DBCommand> executedCommands = new ArrayList<>();
    
    public void addCommand(DBCommand command) {
        commands.add(command);
    }
    
    public void commit() throws Exception {
        System.out.println("\n=== BEGIN TRANSACTION ===");
        try {
            for (DBCommand command : commands) {
                command.execute();
                executedCommands.add(command);
            }
            System.out.println("=== COMMIT ===\n");
        } catch (Exception e) {
            rollback();
            throw e;
        }
    }
    
    public void rollback() {
        System.out.println("\n=== ROLLBACK ===");
        // Undo in reverse order
        for (int i = executedCommands.size() - 1; i >= 0; i--) {
            try {
                executedCommands.get(i).undo();
            } catch (Exception e) {
                System.out.println("Error during rollback: " + e.getMessage());
            }
        }
        executedCommands.clear();
    }
}

// Usage
public class DatabaseApp {
    public static void main(String[] args) throws Exception {
        Database db = new Database();
        
        Transaction tx = new Transaction();
        tx.addCommand(new InsertDBCommand(db, "users", "1", "John"));
        tx.addCommand(new InsertDBCommand(db, "users", "2", "Jane"));
        tx.addCommand(new UpdateDBCommand(db, "users", "1", "John Doe"));
        
        tx.commit();
        
        // Another transaction that we'll rollback
        Transaction tx2 = new Transaction();
        tx2.addCommand(new InsertDBCommand(db, "users", "3", "Bob"));
        tx2.addCommand(new UpdateDBCommand(db, "users", "1", "Johnny"));
        
        tx2.commit();
        
        // Oops, rollback!
        tx2.rollback();
    }
}
```

---

## When to Use Command Pattern

### ✅ Use When:
1. **Queue** operations for later execution
2. **Undo/Redo** functionality needed
3. **Log** operations for audit or recovery
4. **Callback** mechanism needed
5. **Transactions** with rollback

### ❌ Don't Use When:
1. Simple direct method calls are enough
2. No need for undo/log/queue
3. Would add unnecessary complexity

---

## Command vs Other Patterns

| Pattern | Comparison |
|---------|------------|
| **Command** | Encapsulates request with all info to execute |
| **Strategy** | Encapsulates algorithm, interchangeable |
| **Memento** | Captures state (often used with Command for undo) |
| **Chain of Responsibility** | Passes request through chain |

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Intent** | Encapsulate request as object |
| **Key Components** | Command, Receiver, Invoker, Client |
| **Benefits** | Undo/redo, logging, queuing, decoupling |
| **Use Case** | GUI buttons, transactions, task schedulers |

### Remember:
- Command **encapsulates** action + receiver
- Invoker **knows nothing** about receiver
- Commands can be **queued, logged, undone**
- Use **Macro Command** for composite operations

---

**Next: Iterator Pattern →**
