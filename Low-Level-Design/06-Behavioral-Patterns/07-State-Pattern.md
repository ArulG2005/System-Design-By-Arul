# State Design Pattern

## Intent

> **Allow an object to alter its behavior when its internal state changes. The object will appear to change its class.**

---

## The Problem

You have an object with **state-dependent behavior**:
- Different states = different behaviors
- Lots of **if-else/switch** based on state
- Adding new states means changing multiple methods
- State transitions are scattered throughout code

### Bad Code: Giant if-else

```java
// This is TERRIBLE! ❌
class Document {
    private String state = "DRAFT";
    
    public void publish() {
        if (state.equals("DRAFT")) {
            state = "MODERATION";
        } else if (state.equals("MODERATION")) {
            if (currentUser.isAdmin()) {
                state = "PUBLISHED";
            }
        } else if (state.equals("PUBLISHED")) {
            // Can't publish again
        }
        // Adding new state = modify all methods!
    }
    
    public void render() {
        if (state.equals("DRAFT")) {
            // Show edit button
        } else if (state.equals("MODERATION")) {
            // Show pending badge
        } else if (state.equals("PUBLISHED")) {
            // Show public view
        }
    }
}
```

---

## Simple Analogy

Think of a **Traffic Light**:
- **Red state** → Cars stop
- **Yellow state** → Cars slow down
- **Green state** → Cars go

Each state has its own behavior, and the light transitions between states automatically.

Or think of a **Phone**:
- **Locked** → Show lock screen, can only answer calls
- **Unlocked** → Full functionality
- **Low Battery** → Limited features, power saving

---

## Structure

```
┌─────────────────────────────────────┐
│            Context                  │
├─────────────────────────────────────┤
│ - state: State                      │
├─────────────────────────────────────┤        ┌─────────────────────────┐
│ + setState(State)                   │        │     «interface»         │
│ + request()  {                      │───────>│        State            │
│     state.handle(this);             │        ├─────────────────────────┤
│   }                                 │        │ + handle(Context)       │
└─────────────────────────────────────┘        └───────────△─────────────┘
                                                          │
                                      ┌───────────────────┼───────────────────┐
                                      │                   │                   │
                              ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
                              │ ConcreteStateA │   │ ConcreteStateB │   │ ConcreteStateC │
                              ├───────────────┤   ├───────────────┤   ├───────────────┤
                              │ + handle()    │   │ + handle()    │   │ + handle()    │
                              └───────────────┘   └───────────────┘   └───────────────┘
```

---

## Basic Example: Vending Machine

```java
// State interface
interface VendingMachineState {
    void insertMoney(VendingMachine machine, int amount);
    void selectProduct(VendingMachine machine, String product);
    void dispense(VendingMachine machine);
    void cancel(VendingMachine machine);
}

// Context
class VendingMachine {
    private VendingMachineState state;
    private int balance = 0;
    private String selectedProduct;
    private Map<String, Integer> products = new HashMap<>();
    
    public VendingMachine() {
        products.put("Cola", 150);      // $1.50
        products.put("Chips", 100);     // $1.00
        products.put("Candy", 75);      // $0.75
        
        state = new IdleState();  // Initial state
    }
    
    public void setState(VendingMachineState state) {
        this.state = state;
    }
    
    // Delegate to current state
    public void insertMoney(int amount) {
        state.insertMoney(this, amount);
    }
    
    public void selectProduct(String product) {
        state.selectProduct(this, product);
    }
    
    public void dispense() {
        state.dispense(this);
    }
    
    public void cancel() {
        state.cancel(this);
    }
    
    // Internal methods
    public int getBalance() { return balance; }
    public void addBalance(int amount) { balance += amount; }
    public void resetBalance() { balance = 0; }
    
    public String getSelectedProduct() { return selectedProduct; }
    public void setSelectedProduct(String product) { selectedProduct = product; }
    
    public Integer getProductPrice(String product) { return products.get(product); }
    public boolean hasProduct(String product) { return products.containsKey(product); }
}

// Concrete States

// State 1: Idle (waiting for money)
class IdleState implements VendingMachineState {
    @Override
    public void insertMoney(VendingMachine machine, int amount) {
        machine.addBalance(amount);
        System.out.println("Inserted: $" + (amount / 100.0));
        System.out.println("Balance: $" + (machine.getBalance() / 100.0));
        machine.setState(new HasMoneyState());
    }
    
    @Override
    public void selectProduct(VendingMachine machine, String product) {
        System.out.println("Please insert money first");
    }
    
    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("Please insert money and select product");
    }
    
    @Override
    public void cancel(VendingMachine machine) {
        System.out.println("Nothing to cancel");
    }
}

// State 2: Has Money (waiting for selection)
class HasMoneyState implements VendingMachineState {
    @Override
    public void insertMoney(VendingMachine machine, int amount) {
        machine.addBalance(amount);
        System.out.println("Added: $" + (amount / 100.0));
        System.out.println("Balance: $" + (machine.getBalance() / 100.0));
    }
    
    @Override
    public void selectProduct(VendingMachine machine, String product) {
        if (!machine.hasProduct(product)) {
            System.out.println("Product not available: " + product);
            return;
        }
        
        int price = machine.getProductPrice(product);
        if (machine.getBalance() >= price) {
            machine.setSelectedProduct(product);
            System.out.println("Selected: " + product + " ($" + (price / 100.0) + ")");
            machine.setState(new DispensingState());
        } else {
            System.out.println("Not enough balance. Need: $" + (price / 100.0));
        }
    }
    
    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("Please select a product first");
    }
    
    @Override
    public void cancel(VendingMachine machine) {
        int refund = machine.getBalance();
        machine.resetBalance();
        System.out.println("Transaction cancelled. Refund: $" + (refund / 100.0));
        machine.setState(new IdleState());
    }
}

// State 3: Dispensing
class DispensingState implements VendingMachineState {
    @Override
    public void insertMoney(VendingMachine machine, int amount) {
        System.out.println("Please wait, dispensing product...");
    }
    
    @Override
    public void selectProduct(VendingMachine machine, String product) {
        System.out.println("Please wait, dispensing product...");
    }
    
    @Override
    public void dispense(VendingMachine machine) {
        String product = machine.getSelectedProduct();
        int price = machine.getProductPrice(product);
        int change = machine.getBalance() - price;
        
        System.out.println("🎁 Dispensing: " + product);
        
        if (change > 0) {
            System.out.println("Change: $" + (change / 100.0));
        }
        
        machine.resetBalance();
        machine.setSelectedProduct(null);
        machine.setState(new IdleState());
        
        System.out.println("Thank you! Machine ready for next customer.\n");
    }
    
    @Override
    public void cancel(VendingMachine machine) {
        System.out.println("Cannot cancel during dispensing");
    }
}

// Usage
public class VendingMachineDemo {
    public static void main(String[] args) {
        VendingMachine machine = new VendingMachine();
        
        // Scenario 1: Normal purchase
        System.out.println("=== Scenario 1: Normal Purchase ===");
        machine.insertMoney(100);
        machine.insertMoney(100);  // Total $2.00
        machine.selectProduct("Cola");  // $1.50
        machine.dispense();
        
        // Scenario 2: Not enough money
        System.out.println("=== Scenario 2: Not Enough Money ===");
        machine.insertMoney(100);  // $1.00
        machine.selectProduct("Cola");  // Needs $1.50
        machine.insertMoney(50);  // Now $1.50
        machine.selectProduct("Cola");
        machine.dispense();
        
        // Scenario 3: Cancel transaction
        System.out.println("=== Scenario 3: Cancel ===");
        machine.insertMoney(200);
        machine.cancel();
    }
}
```

---

## Real-World Examples

### Example 1: Order Status

```java
// State interface
interface OrderState {
    void next(Order order);
    void prev(Order order);
    void printStatus();
}

// Context
class Order {
    private OrderState state;
    private String orderId;
    
    public Order(String orderId) {
        this.orderId = orderId;
        this.state = new NewState();
    }
    
    public void setState(OrderState state) {
        this.state = state;
    }
    
    public void nextState() {
        state.next(this);
    }
    
    public void prevState() {
        state.prev(this);
    }
    
    public void printStatus() {
        System.out.print("Order " + orderId + ": ");
        state.printStatus();
    }
    
    public String getOrderId() { return orderId; }
}

// States
class NewState implements OrderState {
    @Override
    public void next(Order order) {
        order.setState(new PaidState());
    }
    
    @Override
    public void prev(Order order) {
        System.out.println("Cannot go back from New state - cancelling order");
        order.setState(new CancelledState());
    }
    
    @Override
    public void printStatus() {
        System.out.println("📝 NEW - Waiting for payment");
    }
}

class PaidState implements OrderState {
    @Override
    public void next(Order order) {
        order.setState(new ProcessingState());
    }
    
    @Override
    public void prev(Order order) {
        System.out.println("Refunding payment...");
        order.setState(new NewState());
    }
    
    @Override
    public void printStatus() {
        System.out.println("💳 PAID - Payment received");
    }
}

class ProcessingState implements OrderState {
    @Override
    public void next(Order order) {
        order.setState(new ShippedState());
    }
    
    @Override
    public void prev(Order order) {
        System.out.println("Returning to paid state");
        order.setState(new PaidState());
    }
    
    @Override
    public void printStatus() {
        System.out.println("🔧 PROCESSING - Preparing order");
    }
}

class ShippedState implements OrderState {
    @Override
    public void next(Order order) {
        order.setState(new DeliveredState());
    }
    
    @Override
    public void prev(Order order) {
        System.out.println("Cannot unship - package in transit");
    }
    
    @Override
    public void printStatus() {
        System.out.println("🚚 SHIPPED - In transit");
    }
}

class DeliveredState implements OrderState {
    @Override
    public void next(Order order) {
        System.out.println("Order already delivered - final state");
    }
    
    @Override
    public void prev(Order order) {
        System.out.println("Return process initiated");
        order.setState(new ReturnRequestedState());
    }
    
    @Override
    public void printStatus() {
        System.out.println("✅ DELIVERED - Complete!");
    }
}

class CancelledState implements OrderState {
    @Override
    public void next(Order order) {
        System.out.println("Cannot proceed - order is cancelled");
    }
    
    @Override
    public void prev(Order order) {
        System.out.println("Cannot undo cancellation");
    }
    
    @Override
    public void printStatus() {
        System.out.println("❌ CANCELLED");
    }
}

class ReturnRequestedState implements OrderState {
    @Override
    public void next(Order order) {
        System.out.println("Processing return...");
        order.setState(new CancelledState());
    }
    
    @Override
    public void prev(Order order) {
        System.out.println("Return cancelled - keeping order");
        order.setState(new DeliveredState());
    }
    
    @Override
    public void printStatus() {
        System.out.println("🔄 RETURN REQUESTED");
    }
}

// Usage
public class OrderDemo {
    public static void main(String[] args) {
        Order order = new Order("ORD-001");
        
        order.printStatus();  // NEW
        
        order.nextState();
        order.printStatus();  // PAID
        
        order.nextState();
        order.printStatus();  // PROCESSING
        
        order.nextState();
        order.printStatus();  // SHIPPED
        
        order.nextState();
        order.printStatus();  // DELIVERED
        
        // Customer wants return
        order.prevState();
        order.printStatus();  // RETURN REQUESTED
    }
}
```

---

### Example 2: Document Workflow

```java
// State interface
interface DocumentState {
    void edit(Document doc, String content);
    void review(Document doc);
    void publish(Document doc);
    void reject(Document doc, String reason);
}

// Context
class Document {
    private String content = "";
    private DocumentState state;
    private String author;
    
    public Document(String author) {
        this.author = author;
        this.state = new DraftState();
    }
    
    public void setState(DocumentState state) {
        this.state = state;
        System.out.println("Document state: " + state.getClass().getSimpleName());
    }
    
    public void edit(String content) { state.edit(this, content); }
    public void review() { state.review(this); }
    public void publish() { state.publish(this); }
    public void reject(String reason) { state.reject(this, reason); }
    
    public String getContent() { return content; }
    public void setContent(String content) { this.content = content; }
    public String getAuthor() { return author; }
}

// Draft State
class DraftState implements DocumentState {
    @Override
    public void edit(Document doc, String content) {
        doc.setContent(content);
        System.out.println("Content updated: " + content);
    }
    
    @Override
    public void review(Document doc) {
        if (doc.getContent().isEmpty()) {
            System.out.println("Cannot submit empty document for review");
            return;
        }
        System.out.println("Submitted for review");
        doc.setState(new ReviewState());
    }
    
    @Override
    public void publish(Document doc) {
        System.out.println("Cannot publish draft - submit for review first");
    }
    
    @Override
    public void reject(Document doc, String reason) {
        System.out.println("Document is already in draft");
    }
}

// Review State
class ReviewState implements DocumentState {
    @Override
    public void edit(Document doc, String content) {
        System.out.println("Cannot edit during review - reject to edit");
    }
    
    @Override
    public void review(Document doc) {
        System.out.println("Already in review");
    }
    
    @Override
    public void publish(Document doc) {
        System.out.println("Document approved and published!");
        doc.setState(new PublishedState());
    }
    
    @Override
    public void reject(Document doc, String reason) {
        System.out.println("Document rejected: " + reason);
        doc.setState(new DraftState());
    }
}

// Published State
class PublishedState implements DocumentState {
    @Override
    public void edit(Document doc, String content) {
        System.out.println("Creating new draft version for editing...");
        doc.setState(new DraftState());
        doc.edit(content);
    }
    
    @Override
    public void review(Document doc) {
        System.out.println("Document is already published");
    }
    
    @Override
    public void publish(Document doc) {
        System.out.println("Document is already published");
    }
    
    @Override
    public void reject(Document doc, String reason) {
        System.out.println("Unpublishing document: " + reason);
        doc.setState(new DraftState());
    }
}

// Usage
public class DocumentWorkflow {
    public static void main(String[] args) {
        Document doc = new Document("John");
        
        System.out.println("=== Creating Document ===");
        doc.edit("Hello World");
        doc.edit("Hello World - Updated");
        
        System.out.println("\n=== Submitting for Review ===");
        doc.review();
        doc.edit("Try to edit");  // Should fail
        
        System.out.println("\n=== Rejecting ===");
        doc.reject("Needs more content");
        doc.edit("Hello World - More detailed content here");
        
        System.out.println("\n=== Resubmitting ===");
        doc.review();
        
        System.out.println("\n=== Publishing ===");
        doc.publish();
        
        System.out.println("\n=== Editing Published Doc ===");
        doc.edit("Version 2 content");
    }
}
```

---

### Example 3: Audio Player

```java
// State interface
interface PlayerState {
    void play(AudioPlayer player);
    void pause(AudioPlayer player);
    void stop(AudioPlayer player);
    void next(AudioPlayer player);
    void previous(AudioPlayer player);
}

// Context
class AudioPlayer {
    private PlayerState state;
    private List<String> playlist;
    private int currentTrack = 0;
    
    public AudioPlayer() {
        playlist = Arrays.asList(
            "Song 1 - Artist A",
            "Song 2 - Artist B", 
            "Song 3 - Artist C"
        );
        state = new StoppedState();
    }
    
    public void setState(PlayerState state) {
        this.state = state;
    }
    
    // Delegate to state
    public void play() { state.play(this); }
    public void pause() { state.pause(this); }
    public void stop() { state.stop(this); }
    public void next() { state.next(this); }
    public void previous() { state.previous(this); }
    
    // Internal methods
    public String getCurrentSong() {
        return playlist.get(currentTrack);
    }
    
    public void nextTrack() {
        currentTrack = (currentTrack + 1) % playlist.size();
    }
    
    public void previousTrack() {
        currentTrack = currentTrack > 0 ? currentTrack - 1 : playlist.size() - 1;
    }
    
    public void resetTrack() {
        currentTrack = 0;
    }
}

// Stopped State
class StoppedState implements PlayerState {
    @Override
    public void play(AudioPlayer player) {
        System.out.println("▶️ Playing: " + player.getCurrentSong());
        player.setState(new PlayingState());
    }
    
    @Override
    public void pause(AudioPlayer player) {
        System.out.println("Already stopped");
    }
    
    @Override
    public void stop(AudioPlayer player) {
        System.out.println("Already stopped");
    }
    
    @Override
    public void next(AudioPlayer player) {
        player.nextTrack();
        System.out.println("Next track: " + player.getCurrentSong());
    }
    
    @Override
    public void previous(AudioPlayer player) {
        player.previousTrack();
        System.out.println("Previous track: " + player.getCurrentSong());
    }
}

// Playing State
class PlayingState implements PlayerState {
    @Override
    public void play(AudioPlayer player) {
        System.out.println("Already playing: " + player.getCurrentSong());
    }
    
    @Override
    public void pause(AudioPlayer player) {
        System.out.println("⏸️ Paused: " + player.getCurrentSong());
        player.setState(new PausedState());
    }
    
    @Override
    public void stop(AudioPlayer player) {
        System.out.println("⏹️ Stopped");
        player.resetTrack();
        player.setState(new StoppedState());
    }
    
    @Override
    public void next(AudioPlayer player) {
        player.nextTrack();
        System.out.println("⏭️ Now playing: " + player.getCurrentSong());
    }
    
    @Override
    public void previous(AudioPlayer player) {
        player.previousTrack();
        System.out.println("⏮️ Now playing: " + player.getCurrentSong());
    }
}

// Paused State
class PausedState implements PlayerState {
    @Override
    public void play(AudioPlayer player) {
        System.out.println("▶️ Resuming: " + player.getCurrentSong());
        player.setState(new PlayingState());
    }
    
    @Override
    public void pause(AudioPlayer player) {
        System.out.println("Already paused");
    }
    
    @Override
    public void stop(AudioPlayer player) {
        System.out.println("⏹️ Stopped");
        player.resetTrack();
        player.setState(new StoppedState());
    }
    
    @Override
    public void next(AudioPlayer player) {
        player.nextTrack();
        System.out.println("⏭️ Paused on: " + player.getCurrentSong());
    }
    
    @Override
    public void previous(AudioPlayer player) {
        player.previousTrack();
        System.out.println("⏮️ Paused on: " + player.getCurrentSong());
    }
}

// Usage
public class MusicApp {
    public static void main(String[] args) {
        AudioPlayer player = new AudioPlayer();
        
        player.play();     // Start playing
        player.next();     // Next song while playing
        player.pause();    // Pause
        player.previous(); // Previous while paused
        player.play();     // Resume
        player.stop();     // Stop
        player.pause();    // Can't pause when stopped
    }
}
```

---

## State vs Strategy

| State | Strategy |
|-------|----------|
| Object changes behavior based on **internal state** | Object uses **different algorithms** |
| States know about and **transition to** each other | Strategies are **independent** |
| State can change during object's lifetime | Strategy typically set once (or rarely changes) |
| **Finite state machine** pattern | **Algorithm selection** pattern |

---

## When to Use State Pattern

### ✅ Use When:
1. Object behavior **depends on state**
2. You have **lots of conditional code** checking state
3. State transitions follow **defined rules**
4. States are **finite and well-defined**

### ❌ Don't Use When:
1. Only 2-3 simple states
2. State-conditional logic is minimal
3. Would add unnecessary complexity

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Intent** | Change object behavior based on internal state |
| **Key Idea** | Encapsulate states as separate classes |
| **Benefits** | Eliminates if-else, clean transitions, SRP |
| **Use Case** | Vending machines, workflows, games, media players |

### Remember:
- Context delegates to current **state object**
- States can trigger **transitions** to other states
- Each state **encapsulates** its own behavior
- Adding new state = adding new class (OCP!)

---

**Next: Chain of Responsibility Pattern →**
