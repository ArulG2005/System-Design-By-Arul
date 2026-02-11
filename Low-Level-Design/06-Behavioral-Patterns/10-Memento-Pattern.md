# Memento Design Pattern

## Intent

> **Without violating encapsulation, capture and externalize an object's internal state so that the object can be restored to this state later.**

---

## The Problem

You need to implement **undo/redo** or **rollback** functionality:
- Need to save object's **internal state**
- Don't want to expose **private fields** (encapsulation)
- Need to restore to **previous states**

### Bad Approach: Exposing internals

```java
// BAD! Breaks encapsulation ❌
class TextEditor {
    private String content;
    private int cursorPosition;
    
    // Exposing private state!
    public String getContent() { return content; }
    public int getCursorPosition() { return cursorPosition; }
    
    // Anyone can mess with internal state!
    public void setContent(String c) { content = c; }
    public void setCursorPosition(int p) { cursorPosition = p; }
}
```

---

## Simple Analogy

Think of **Ctrl+Z (Undo)** in any application:
- Before each action, state is **saved**
- Undo **restores** to previous state
- Multiple undos = restore to **any previous** state

Or think of **Game Save Points**:
- Save game at checkpoint
- If you die, restore from save
- State (health, position, inventory) restored

---

## Structure

```
┌─────────────────────────────────────────┐
│            Originator                    │
├─────────────────────────────────────────┤
│ - state: String                         │
├─────────────────────────────────────────┤
│ + save(): Memento                       │──────────────────┐
│ + restore(Memento)                      │                  │
└─────────────────────────────────────────┘                  │
                                                             ▼
                                              ┌─────────────────────────┐
┌───────────────────────────────────────┐     │        Memento          │
│          Caretaker                     │     ├─────────────────────────┤
├───────────────────────────────────────┤     │ - state: String         │
│ - history: List<Memento>              │◄────┤ + getState(): String    │
├───────────────────────────────────────┤     │   (private to          │
│ + backup()                            │     │    Originator)          │
│ + undo()                              │     └─────────────────────────┘
│ + redo()                              │     
└───────────────────────────────────────┘     

Key: Memento is opaque to Caretaker (can't access state)
```

---

## Basic Example: Text Editor

```java
// Memento - Holds the state
class TextMemento {
    private final String content;
    private final int cursorPosition;
    private final LocalDateTime timestamp;
    
    public TextMemento(String content, int cursorPosition) {
        this.content = content;
        this.cursorPosition = cursorPosition;
        this.timestamp = LocalDateTime.now();
    }
    
    // Package-private or inner class access
    String getContent() { return content; }
    int getCursorPosition() { return cursorPosition; }
    
    public LocalDateTime getTimestamp() { return timestamp; }
}

// Originator - Creates and restores from mementos
class TextEditor {
    private StringBuilder content;
    private int cursorPosition;
    
    public TextEditor() {
        content = new StringBuilder();
        cursorPosition = 0;
    }
    
    public void type(String text) {
        content.insert(cursorPosition, text);
        cursorPosition += text.length();
        System.out.println("Typed: \"" + text + "\"");
    }
    
    public void delete(int count) {
        if (cursorPosition >= count) {
            int start = cursorPosition - count;
            content.delete(start, cursorPosition);
            cursorPosition = start;
            System.out.println("Deleted " + count + " characters");
        }
    }
    
    public void moveCursor(int position) {
        if (position >= 0 && position <= content.length()) {
            cursorPosition = position;
        }
    }
    
    // Create memento
    public TextMemento save() {
        System.out.println("  [Saving state...]");
        return new TextMemento(content.toString(), cursorPosition);
    }
    
    // Restore from memento
    public void restore(TextMemento memento) {
        content = new StringBuilder(memento.getContent());
        cursorPosition = memento.getCursorPosition();
        System.out.println("  [Restored to previous state]");
    }
    
    public void display() {
        String text = content.toString();
        String cursorLine = " ".repeat(cursorPosition) + "^";
        System.out.println("Content: \"" + text + "\"");
        System.out.println("Cursor:  " + cursorLine);
    }
}

// Caretaker - Manages mementos
class History {
    private List<TextMemento> undoStack = new ArrayList<>();
    private List<TextMemento> redoStack = new ArrayList<>();
    private TextEditor editor;
    
    public History(TextEditor editor) {
        this.editor = editor;
    }
    
    public void backup() {
        undoStack.add(editor.save());
        redoStack.clear();  // Clear redo after new action
    }
    
    public void undo() {
        if (undoStack.isEmpty()) {
            System.out.println("Nothing to undo!");
            return;
        }
        
        // Save current state for redo
        redoStack.add(editor.save());
        
        // Restore previous state
        TextMemento memento = undoStack.remove(undoStack.size() - 1);
        editor.restore(memento);
    }
    
    public void redo() {
        if (redoStack.isEmpty()) {
            System.out.println("Nothing to redo!");
            return;
        }
        
        // Save current for undo
        undoStack.add(editor.save());
        
        // Restore redo state
        TextMemento memento = redoStack.remove(redoStack.size() - 1);
        editor.restore(memento);
    }
    
    public void showHistory() {
        System.out.println("\n--- History ---");
        System.out.println("Undo stack: " + undoStack.size() + " states");
        System.out.println("Redo stack: " + redoStack.size() + " states");
    }
}

// Usage
public class TextEditorDemo {
    public static void main(String[] args) {
        TextEditor editor = new TextEditor();
        History history = new History(editor);
        
        // Initial state
        history.backup();
        
        // User types
        editor.type("Hello");
        editor.display();
        history.backup();
        
        editor.type(" World");
        editor.display();
        history.backup();
        
        editor.type("!");
        editor.display();
        
        history.showHistory();
        
        // Undo!
        System.out.println("\n=== Undo ===");
        history.undo();
        editor.display();
        
        System.out.println("\n=== Undo Again ===");
        history.undo();
        editor.display();
        
        System.out.println("\n=== Redo ===");
        history.redo();
        editor.display();
    }
}
```

---

## Real-World Examples

### Example 1: Game Save System

```java
// Game State (Memento)
class GameSave {
    private final int playerHealth;
    private final int playerMana;
    private final String currentLevel;
    private final List<String> inventory;
    private final Map<String, Boolean> achievements;
    private final LocalDateTime saveTime;
    private final String saveName;
    
    public GameSave(int health, int mana, String level, 
                    List<String> inventory, Map<String, Boolean> achievements,
                    String saveName) {
        this.playerHealth = health;
        this.playerMana = mana;
        this.currentLevel = level;
        this.inventory = new ArrayList<>(inventory);
        this.achievements = new HashMap<>(achievements);
        this.saveTime = LocalDateTime.now();
        this.saveName = saveName;
    }
    
    // Getters (package-private for Originator access)
    int getPlayerHealth() { return playerHealth; }
    int getPlayerMana() { return playerMana; }
    String getCurrentLevel() { return currentLevel; }
    List<String> getInventory() { return new ArrayList<>(inventory); }
    Map<String, Boolean> getAchievements() { return new HashMap<>(achievements); }
    
    public String getSaveName() { return saveName; }
    public LocalDateTime getSaveTime() { return saveTime; }
    
    @Override
    public String toString() {
        return String.format("%s (Level: %s, HP: %d) - %s", 
            saveName, currentLevel, playerHealth, saveTime);
    }
}

// Game (Originator)
class Game {
    private int playerHealth = 100;
    private int playerMana = 50;
    private String currentLevel = "Level 1 - Village";
    private List<String> inventory = new ArrayList<>();
    private Map<String, Boolean> achievements = new HashMap<>();
    
    public Game() {
        inventory.add("Wooden Sword");
        achievements.put("First Steps", false);
        achievements.put("Dragon Slayer", false);
    }
    
    // Game actions
    public void takeDamage(int damage) {
        playerHealth = Math.max(0, playerHealth - damage);
        System.out.println("💔 Took " + damage + " damage! HP: " + playerHealth);
        if (playerHealth == 0) {
            System.out.println("☠️ GAME OVER!");
        }
    }
    
    public void heal(int amount) {
        playerHealth = Math.min(100, playerHealth + amount);
        System.out.println("💚 Healed " + amount + "! HP: " + playerHealth);
    }
    
    public void useMana(int amount) {
        if (playerMana >= amount) {
            playerMana -= amount;
            System.out.println("✨ Used " + amount + " mana. Remaining: " + playerMana);
        }
    }
    
    public void collectItem(String item) {
        inventory.add(item);
        System.out.println("📦 Collected: " + item);
    }
    
    public void goToLevel(String level) {
        currentLevel = level;
        System.out.println("🗺️ Entered: " + level);
    }
    
    public void unlockAchievement(String name) {
        if (achievements.containsKey(name)) {
            achievements.put(name, true);
            System.out.println("🏆 Achievement Unlocked: " + name);
        }
    }
    
    // Memento operations
    public GameSave save(String saveName) {
        System.out.println("💾 Game saved: " + saveName);
        return new GameSave(playerHealth, playerMana, currentLevel, 
                           inventory, achievements, saveName);
    }
    
    public void load(GameSave save) {
        playerHealth = save.getPlayerHealth();
        playerMana = save.getPlayerMana();
        currentLevel = save.getCurrentLevel();
        inventory = save.getInventory();
        achievements = save.getAchievements();
        System.out.println("📂 Game loaded: " + save.getSaveName());
    }
    
    public void status() {
        System.out.println("\n=== Game Status ===");
        System.out.println("Level: " + currentLevel);
        System.out.println("HP: " + playerHealth + " | Mana: " + playerMana);
        System.out.println("Inventory: " + inventory);
        System.out.println("==================\n");
    }
}

// Save Manager (Caretaker)
class SaveManager {
    private Map<String, GameSave> saves = new HashMap<>();
    private List<GameSave> autoSaves = new ArrayList<>();
    private static final int MAX_AUTOSAVES = 5;
    
    public void manualSave(Game game, String name) {
        saves.put(name, game.save(name));
    }
    
    public void autoSave(Game game) {
        String name = "AutoSave-" + (autoSaves.size() + 1);
        GameSave save = game.save(name);
        autoSaves.add(save);
        
        // Keep only last 5 autosaves
        if (autoSaves.size() > MAX_AUTOSAVES) {
            autoSaves.remove(0);
        }
    }
    
    public void loadSave(Game game, String name) {
        GameSave save = saves.get(name);
        if (save != null) {
            game.load(save);
        } else {
            System.out.println("Save not found: " + name);
        }
    }
    
    public void loadLastAutoSave(Game game) {
        if (!autoSaves.isEmpty()) {
            game.load(autoSaves.get(autoSaves.size() - 1));
        } else {
            System.out.println("No autosaves available!");
        }
    }
    
    public void showSaves() {
        System.out.println("\n📁 SAVE FILES:");
        System.out.println("Manual saves:");
        saves.values().forEach(s -> System.out.println("  - " + s));
        System.out.println("Auto saves:");
        autoSaves.forEach(s -> System.out.println("  - " + s));
    }
}

// Usage
public class GameDemo {
    public static void main(String[] args) {
        Game game = new Game();
        SaveManager saveManager = new SaveManager();
        
        game.status();
        
        // Play and save
        game.goToLevel("Level 2 - Forest");
        game.collectItem("Health Potion");
        game.unlockAchievement("First Steps");
        saveManager.autoSave(game);
        
        game.goToLevel("Level 3 - Dragon Cave");
        game.collectItem("Dragon Scale Shield");
        saveManager.manualSave(game, "Before Boss");
        
        game.status();
        
        // Fight boss!
        System.out.println("=== BOSS FIGHT ===");
        game.takeDamage(50);
        game.useMana(30);
        game.takeDamage(40);
        game.takeDamage(20);  // Dead!
        
        game.status();
        
        // Load save
        saveManager.showSaves();
        System.out.println("\n=== Loading Save ===");
        saveManager.loadSave(game, "Before Boss");
        game.status();
    }
}
```

---

### Example 2: Transaction Rollback

```java
// Account State (Memento)
class AccountState {
    private final double balance;
    private final List<String> transactions;
    private final LocalDateTime timestamp;
    
    public AccountState(double balance, List<String> transactions) {
        this.balance = balance;
        this.transactions = new ArrayList<>(transactions);
        this.timestamp = LocalDateTime.now();
    }
    
    double getBalance() { return balance; }
    List<String> getTransactions() { return new ArrayList<>(transactions); }
    LocalDateTime getTimestamp() { return timestamp; }
}

// Bank Account (Originator)
class BankAccount {
    private String accountNumber;
    private double balance;
    private List<String> transactions = new ArrayList<>();
    
    public BankAccount(String accountNumber, double initialBalance) {
        this.accountNumber = accountNumber;
        this.balance = initialBalance;
        transactions.add("Initial deposit: $" + initialBalance);
    }
    
    public void deposit(double amount) throws Exception {
        if (amount <= 0) {
            throw new Exception("Invalid deposit amount");
        }
        balance += amount;
        transactions.add("Deposit: +$" + amount);
        System.out.println("✅ Deposited $" + amount);
    }
    
    public void withdraw(double amount) throws Exception {
        if (amount > balance) {
            throw new Exception("Insufficient funds");
        }
        balance -= amount;
        transactions.add("Withdrawal: -$" + amount);
        System.out.println("✅ Withdrew $" + amount);
    }
    
    public void transfer(BankAccount target, double amount) throws Exception {
        if (amount > balance) {
            throw new Exception("Insufficient funds for transfer");
        }
        balance -= amount;
        target.balance += amount;
        
        transactions.add("Transfer to " + target.accountNumber + ": -$" + amount);
        target.transactions.add("Transfer from " + accountNumber + ": +$" + amount);
        
        System.out.println("✅ Transferred $" + amount + " to " + target.accountNumber);
    }
    
    // Memento operations
    public AccountState saveState() {
        return new AccountState(balance, transactions);
    }
    
    public void restoreState(AccountState state) {
        this.balance = state.getBalance();
        this.transactions = state.getTransactions();
        System.out.println("🔄 Account state restored for " + accountNumber);
    }
    
    public void printStatement() {
        System.out.println("\n--- Account: " + accountNumber + " ---");
        System.out.println("Balance: $" + balance);
        System.out.println("Transactions:");
        transactions.forEach(t -> System.out.println("  " + t));
    }
}

// Transaction Manager (Caretaker)
class TransactionManager {
    private Map<String, AccountState> savepoints = new HashMap<>();
    
    public void beginTransaction(String name, BankAccount... accounts) {
        System.out.println("\n💼 BEGIN TRANSACTION: " + name);
        for (BankAccount account : accounts) {
            savepoints.put(name + "-" + account.hashCode(), account.saveState());
        }
    }
    
    public void commit(String name) {
        // Remove savepoints - transaction successful
        savepoints.entrySet().removeIf(e -> e.getKey().startsWith(name));
        System.out.println("✅ TRANSACTION COMMITTED: " + name);
    }
    
    public void rollback(String name, BankAccount... accounts) {
        System.out.println("⚠️ ROLLING BACK: " + name);
        for (BankAccount account : accounts) {
            String key = name + "-" + account.hashCode();
            AccountState savedState = savepoints.get(key);
            if (savedState != null) {
                account.restoreState(savedState);
            }
        }
        // Clean up savepoints
        savepoints.entrySet().removeIf(e -> e.getKey().startsWith(name));
    }
}

// Usage
public class BankDemo {
    public static void main(String[] args) {
        BankAccount account1 = new BankAccount("ACC-001", 1000);
        BankAccount account2 = new BankAccount("ACC-002", 500);
        TransactionManager txManager = new TransactionManager();
        
        account1.printStatement();
        account2.printStatement();
        
        // Successful transaction
        try {
            txManager.beginTransaction("TX-001", account1, account2);
            account1.transfer(account2, 200);
            txManager.commit("TX-001");
        } catch (Exception e) {
            txManager.rollback("TX-001", account1, account2);
        }
        
        account1.printStatement();
        account2.printStatement();
        
        // Failed transaction - will rollback
        try {
            txManager.beginTransaction("TX-002", account1, account2);
            account1.withdraw(300);
            account1.transfer(account2, 2000);  // This will fail!
            txManager.commit("TX-002");
        } catch (Exception e) {
            System.out.println("\n❌ Transaction failed: " + e.getMessage());
            txManager.rollback("TX-002", account1, account2);
        }
        
        account1.printStatement();
        account2.printStatement();
    }
}
```

---

### Example 3: Form State with Version History

```java
// Form Snapshot (Memento)
class FormSnapshot {
    private final Map<String, String> fieldValues;
    private final int version;
    private final LocalDateTime timestamp;
    private final String description;
    
    public FormSnapshot(Map<String, String> values, int version, String description) {
        this.fieldValues = new HashMap<>(values);
        this.version = version;
        this.timestamp = LocalDateTime.now();
        this.description = description;
    }
    
    Map<String, String> getFieldValues() { return new HashMap<>(fieldValues); }
    public int getVersion() { return version; }
    public String getDescription() { return description; }
    public LocalDateTime getTimestamp() { return timestamp; }
    
    @Override
    public String toString() {
        return String.format("v%d - %s (%s)", version, description, 
            timestamp.format(DateTimeFormatter.ofPattern("HH:mm:ss")));
    }
}

// Form (Originator)
class Form {
    private Map<String, String> fields = new LinkedHashMap<>();
    private int currentVersion = 0;
    
    public void setField(String name, String value) {
        fields.put(name, value);
        System.out.println("Set " + name + " = " + value);
    }
    
    public String getField(String name) {
        return fields.get(name);
    }
    
    public void clearField(String name) {
        fields.remove(name);
    }
    
    public FormSnapshot createSnapshot(String description) {
        currentVersion++;
        System.out.println("📸 Creating snapshot: v" + currentVersion + " - " + description);
        return new FormSnapshot(fields, currentVersion, description);
    }
    
    public void restore(FormSnapshot snapshot) {
        fields = snapshot.getFieldValues();
        System.out.println("🔄 Restored to: " + snapshot);
    }
    
    public void display() {
        System.out.println("\n📋 Form Contents:");
        if (fields.isEmpty()) {
            System.out.println("  (empty)");
        }
        fields.forEach((k, v) -> System.out.println("  " + k + ": " + v));
    }
}

// History Manager (Caretaker)
class FormHistory {
    private List<FormSnapshot> versions = new ArrayList<>();
    private int currentIndex = -1;
    
    public void save(FormSnapshot snapshot) {
        // Remove any "future" versions if we saved after undo
        while (versions.size() > currentIndex + 1) {
            versions.remove(versions.size() - 1);
        }
        versions.add(snapshot);
        currentIndex = versions.size() - 1;
    }
    
    public FormSnapshot undo() {
        if (currentIndex > 0) {
            currentIndex--;
            System.out.println("↩️ Undo to: " + versions.get(currentIndex));
            return versions.get(currentIndex);
        }
        System.out.println("Nothing to undo!");
        return null;
    }
    
    public FormSnapshot redo() {
        if (currentIndex < versions.size() - 1) {
            currentIndex++;
            System.out.println("↪️ Redo to: " + versions.get(currentIndex));
            return versions.get(currentIndex);
        }
        System.out.println("Nothing to redo!");
        return null;
    }
    
    public FormSnapshot getVersion(int version) {
        for (FormSnapshot snapshot : versions) {
            if (snapshot.getVersion() == version) {
                return snapshot;
            }
        }
        return null;
    }
    
    public void showHistory() {
        System.out.println("\n📜 VERSION HISTORY:");
        for (int i = 0; i < versions.size(); i++) {
            String marker = (i == currentIndex) ? " ◄ current" : "";
            System.out.println("  " + versions.get(i) + marker);
        }
    }
}

// Usage
public class FormDemo {
    public static void main(String[] args) {
        Form form = new Form();
        FormHistory history = new FormHistory();
        
        // Fill form - save versions
        form.setField("Name", "John");
        form.setField("Email", "john@email.com");
        history.save(form.createSnapshot("Initial entry"));
        
        form.setField("Phone", "123-456-7890");
        form.setField("Address", "123 Main St");
        history.save(form.createSnapshot("Added contact info"));
        
        form.setField("Name", "John Doe");
        form.setField("Company", "Tech Corp");
        history.save(form.createSnapshot("Updated name, added company"));
        
        form.display();
        history.showHistory();
        
        // Undo
        System.out.println("\n=== UNDO ===");
        FormSnapshot prev = history.undo();
        if (prev != null) form.restore(prev);
        form.display();
        
        System.out.println("\n=== UNDO AGAIN ===");
        prev = history.undo();
        if (prev != null) form.restore(prev);
        form.display();
        
        System.out.println("\n=== REDO ===");
        FormSnapshot next = history.redo();
        if (next != null) form.restore(next);
        form.display();
        
        history.showHistory();
    }
}
```

---

## When to Use

### ✅ Use When:
1. Need **undo/redo** functionality
2. Need to save **snapshots** of object state
3. Want to maintain **encapsulation**
4. Need **transaction rollback**

### ❌ Don't Use When:
1. **Frequent** saves with large state (memory issue)
2. Object state is **trivial** (simple values)
3. State changes are **irreversible** by design

---

## Memory Considerations

```java
// Option 1: Limit history size
class LimitedHistory {
    private static final int MAX_SIZE = 50;
    private LinkedList<Memento> history = new LinkedList<>();
    
    public void save(Memento m) {
        if (history.size() >= MAX_SIZE) {
            history.removeFirst();  // Remove oldest
        }
        history.addLast(m);
    }
}

// Option 2: Incremental mementos (store only changes)
class IncrementalMemento {
    private Map<String, Object> changedFields;  // Only what changed
    private IncrementalMemento previous;
    
    // Reconstruct full state by traversing chain
}

// Option 3: Compress/serialize large states
class CompressedMemento {
    private byte[] compressedState;
    
    public CompressedMemento(Object state) {
        this.compressedState = compress(serialize(state));
    }
}
```

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Intent** | Save and restore object state |
| **Key Idea** | Memento holds state, only Originator can access it |
| **Participants** | Originator, Memento, Caretaker |
| **Benefits** | Preserves encapsulation, enables undo/redo |
| **Use Cases** | Text editors, games, transactions, version control |

### Remember:
- **Originator** creates mementos (only it knows internal state)
- **Memento** is **opaque** to Caretaker
- **Caretaker** stores mementos (doesn't peek inside)
- Watch out for **memory usage** with large states!

---

**Next: Visitor Pattern →**
