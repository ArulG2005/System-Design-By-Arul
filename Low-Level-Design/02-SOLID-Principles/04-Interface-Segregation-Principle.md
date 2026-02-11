# Interface Segregation Principle (ISP)

## The Fourth SOLID Principle

> **"Clients should not be forced to depend on interfaces they do not use."**
> — Robert C. Martin

---

## What Does This Mean?

Don't create big, fat interfaces. Instead, create small, focused interfaces.

A class should only need to know about methods it actually uses.

---

## Simple Analogy

Think of a **TV remote**:
- You have a simple TV → You need: Power, Volume, Channel
- You have a smart TV → You need: Power, Volume, Channel + Netflix, YouTube, Smart features

**Bad design**: One remote with 100 buttons for ALL TVs
- Simple TV users confused by YouTube button
- Most buttons unused

**Good design**: 
- Basic remote for simple TVs
- Smart remote for smart TVs (maybe extends basic remote)

---

## Real-World Example: Worker Interface

### ❌ BAD: Fat Interface (Violating ISP)

```java
// One big interface for ALL workers
interface Worker {
    void work();
    void eat();
    void sleep();
    void code();
    void design();
    void manage();
    void test();
    void attendMeeting();
    void writeDocumentation();
}

// Human worker - can do many things
class HumanWorker implements Worker {
    @Override public void work() { System.out.println("Working..."); }
    @Override public void eat() { System.out.println("Eating lunch..."); }
    @Override public void sleep() { System.out.println("Sleeping..."); }
    @Override public void code() { System.out.println("Coding..."); }
    @Override public void design() { System.out.println("Designing..."); }
    @Override public void manage() { System.out.println("Managing team..."); }
    @Override public void test() { System.out.println("Testing..."); }
    @Override public void attendMeeting() { System.out.println("In meeting..."); }
    @Override public void writeDocumentation() { System.out.println("Writing docs..."); }
}

// Robot worker - can't eat or sleep!
class RobotWorker implements Worker {
    @Override public void work() { System.out.println("Robot working..."); }
    
    @Override public void eat() { 
        // Robots don't eat! What do we do?
        throw new UnsupportedOperationException("Robots don't eat!");
    }
    
    @Override public void sleep() { 
        // Robots don't sleep!
        throw new UnsupportedOperationException("Robots don't sleep!");
    }
    
    @Override public void code() { System.out.println("Robot coding..."); }
    @Override public void design() { /* Empty - robots don't design */ }
    @Override public void manage() { 
        throw new UnsupportedOperationException("Robots don't manage!");
    }
    @Override public void test() { System.out.println("Robot testing..."); }
    @Override public void attendMeeting() { /* Empty */ }
    @Override public void writeDocumentation() { /* Empty */ }
}
```

### Problems:
1. RobotWorker is forced to implement methods it doesn't use
2. Throwing exceptions = violates LSP too!
3. Empty implementations = code smell
4. Changes to interface affect ALL implementers

---

### ✅ GOOD: Segregated Interfaces (Following ISP)

```java
// Small, focused interfaces
interface Workable {
    void work();
}

interface Eatable {
    void eat();
}

interface Sleepable {
    void sleep();
}

interface Codeable {
    void code();
}

interface Designable {
    void design();
}

interface Manageable {
    void manage();
}

interface Testable {
    void test();
}

// Human implements only what they need
class HumanWorker implements Workable, Eatable, Sleepable, Codeable, Testable {
    @Override public void work() { System.out.println("Working..."); }
    @Override public void eat() { System.out.println("Eating lunch..."); }
    @Override public void sleep() { System.out.println("Sleeping..."); }
    @Override public void code() { System.out.println("Coding..."); }
    @Override public void test() { System.out.println("Testing..."); }
}

// Manager implements what managers do
class Manager implements Workable, Eatable, Sleepable, Manageable {
    @Override public void work() { System.out.println("Working..."); }
    @Override public void eat() { System.out.println("Eating..."); }
    @Override public void sleep() { System.out.println("Sleeping..."); }
    @Override public void manage() { System.out.println("Managing team..."); }
}

// Robot implements only what robots can do
class RobotWorker implements Workable, Codeable, Testable {
    @Override public void work() { System.out.println("Robot working 24/7..."); }
    @Override public void code() { System.out.println("Robot coding..."); }
    @Override public void test() { System.out.println("Robot testing..."); }
    // No eat, sleep, manage - Robots don't do those!
}
```

---

## Another Example: Printer/Scanner

### ❌ BAD: Fat Interface

```java
// One interface for all machines
interface Machine {
    void print(Document doc);
    void scan(Document doc);
    void fax(Document doc);
    void photocopy(Document doc);
}

// Old printer - can ONLY print!
class OldPrinter implements Machine {
    @Override
    public void print(Document doc) {
        System.out.println("Printing: " + doc.getTitle());
    }
    
    @Override
    public void scan(Document doc) {
        throw new UnsupportedOperationException("Cannot scan!");
    }
    
    @Override
    public void fax(Document doc) {
        throw new UnsupportedOperationException("Cannot fax!");
    }
    
    @Override
    public void photocopy(Document doc) {
        throw new UnsupportedOperationException("Cannot photocopy!");
    }
}

// Modern multi-function printer - can do everything
class ModernPrinter implements Machine {
    @Override public void print(Document doc) { /* ... */ }
    @Override public void scan(Document doc) { /* ... */ }
    @Override public void fax(Document doc) { /* ... */ }
    @Override public void photocopy(Document doc) { /* ... */ }
}
```

---

### ✅ GOOD: Segregated Interfaces

```java
// Separate interfaces for each capability
interface Printer {
    void print(Document doc);
}

interface Scanner {
    void scan(Document doc);
}

interface Fax {
    void fax(Document doc);
}

// Old printer - implements only what it can do
class OldPrinter implements Printer {
    @Override
    public void print(Document doc) {
        System.out.println("Printing: " + doc.getTitle());
    }
    // No scan, fax, or photocopy - it can't do those!
}

// Modern all-in-one printer - implements all interfaces
class ModernPrinter implements Printer, Scanner, Fax {
    @Override
    public void print(Document doc) {
        System.out.println("Modern printing: " + doc.getTitle());
    }
    
    @Override
    public void scan(Document doc) {
        System.out.println("Scanning: " + doc.getTitle());
    }
    
    @Override
    public void fax(Document doc) {
        System.out.println("Faxing: " + doc.getTitle());
    }
}

// Just a scanner
class SimpleScanner implements Scanner {
    @Override
    public void scan(Document doc) {
        System.out.println("Scanning: " + doc.getTitle());
    }
}
```

### Usage:

```java
// Client that only needs printing
class PrintClient {
    private Printer printer;
    
    public PrintClient(Printer printer) {
        this.printer = printer;  // Works with ANY printer!
    }
    
    public void printDocument(Document doc) {
        printer.print(doc);
    }
}

// Can use old printer OR modern printer
PrintClient client1 = new PrintClient(new OldPrinter());
PrintClient client2 = new PrintClient(new ModernPrinter());
```

---

## Another Example: User Management

### ❌ BAD: Fat Interface

```java
interface UserService {
    // Authentication
    User login(String username, String password);
    void logout(User user);
    void resetPassword(String email);
    
    // User CRUD
    User createUser(String name, String email);
    User getUser(int id);
    void updateUser(User user);
    void deleteUser(int id);
    
    // Profile
    void uploadProfilePicture(User user, byte[] image);
    byte[] getProfilePicture(User user);
    
    // Notifications
    void sendNotification(User user, String message);
    List<Notification> getNotifications(User user);
    
    // Admin
    List<User> getAllUsers();
    void banUser(int id);
    void unbanUser(int id);
}

// Login page only needs login/logout
// But has to depend on entire UserService!
class LoginController {
    private UserService userService;  // Has access to deleteUser, banUser, etc.!
    
    public void login(String username, String password) {
        userService.login(username, password);
    }
}
```

---

### ✅ GOOD: Segregated Interfaces

```java
// Authentication operations
interface AuthService {
    User login(String username, String password);
    void logout(User user);
    void resetPassword(String email);
}

// Basic user operations
interface UserCrudService {
    User createUser(String name, String email);
    User getUser(int id);
    void updateUser(User user);
    void deleteUser(int id);
}

// Profile operations
interface ProfileService {
    void uploadProfilePicture(User user, byte[] image);
    byte[] getProfilePicture(User user);
    void updateProfile(User user, ProfileData data);
}

// Notification operations
interface NotificationService {
    void sendNotification(User user, String message);
    List<Notification> getNotifications(User user);
    void markAsRead(int notificationId);
}

// Admin operations
interface AdminService {
    List<User> getAllUsers();
    void banUser(int id);
    void unbanUser(int id);
    UserStats getStats();
}

// Login only depends on what it needs
class LoginController {
    private AuthService authService;  // Only authentication!
    
    public LoginController(AuthService authService) {
        this.authService = authService;
    }
    
    public User login(String username, String password) {
        return authService.login(username, password);
    }
}

// Admin panel depends on admin features
class AdminController {
    private AdminService adminService;
    private UserCrudService userService;
    
    public AdminController(AdminService adminService, UserCrudService userService) {
        this.adminService = adminService;
        this.userService = userService;
    }
    
    public List<User> listAllUsers() {
        return adminService.getAllUsers();
    }
    
    public void banUser(int id) {
        adminService.banUser(id);
    }
}
```

---

## Role Interfaces Pattern

A powerful way to apply ISP:

```java
// Define roles as interfaces
interface Readable {
    String read();
}

interface Writable {
    void write(String data);
}

interface Closeable {
    void close();
}

interface Seekable {
    void seek(long position);
}

// File systems implement what they support
class RegularFile implements Readable, Writable, Closeable, Seekable {
    @Override public String read() { /* ... */ }
    @Override public void write(String data) { /* ... */ }
    @Override public void close() { /* ... */ }
    @Override public void seek(long position) { /* ... */ }
}

class ReadOnlyFile implements Readable, Closeable {
    @Override public String read() { /* ... */ }
    @Override public void close() { /* ... */ }
    // Cannot write or seek
}

class NetworkStream implements Readable, Writable, Closeable {
    @Override public String read() { /* ... */ }
    @Override public void write(String data) { /* ... */ }
    @Override public void close() { /* ... */ }
    // Cannot seek in a network stream
}

// Clients depend only on what they need
class FileReader {
    public void read(Readable readable) {
        String content = readable.read();
        // Process content
    }
}

class FileWriter {
    public void write(Writable writable, String data) {
        writable.write(data);
    }
}
```

---

## Java Examples of ISP

### Java Collections Framework

```java
// Many small interfaces instead of one huge interface

interface Iterable<T> {
    Iterator<T> iterator();
}

interface Collection<E> extends Iterable<E> {
    int size();
    boolean isEmpty();
    boolean contains(Object o);
    boolean add(E e);
    boolean remove(Object o);
    // ...
}

interface List<E> extends Collection<E> {
    E get(int index);
    E set(int index, E element);
    void add(int index, E element);
    // ...
}

interface Set<E> extends Collection<E> {
    // Same as Collection (no duplicates)
}

interface Queue<E> extends Collection<E> {
    E peek();
    E poll();
    boolean offer(E e);
}
```

### Java I/O

```java
// Separate interfaces for different capabilities

interface Readable {
    int read(CharBuffer cb);
}

interface Appendable {
    Appendable append(char c);
    Appendable append(CharSequence csq);
}

interface Closeable {
    void close();
}

interface Flushable {
    void flush();
}

// Classes implement what they need
class FileWriter implements Appendable, Closeable, Flushable { }
class StringReader implements Readable, Closeable { }
```

---

## How to Identify ISP Violations

### Red Flags 🚩

1. **Empty method implementations**
```java
@Override
public void unusedMethod() {
    // Do nothing
}
```

2. **Throwing UnsupportedOperationException**
```java
@Override
public void scan() {
    throw new UnsupportedOperationException();
}
```

3. **Interfaces with many unrelated methods**
```java
interface DoEverything {
    void doA();
    void doB();
    void doC();
    void doX();
    void doY();
    void doZ();
    // 20 more methods...
}
```

4. **Classes implementing interfaces but not using all methods**

5. **Interfaces changed frequently, affecting many classes**

---

## When to Split Interfaces

### Split When:
- Different clients need different methods
- Some implementations leave methods empty
- Interface has more than 5-7 methods
- Methods fall into logical groups

### Don't Split When:
- All methods are highly related
- All implementations use all methods
- Splitting creates too many tiny interfaces
- It makes the code harder to understand

---

## Interface Composition

You can combine small interfaces when needed:

```java
// Small focused interfaces
interface Readable { String read(); }
interface Writable { void write(String data); }
interface Closeable { void close(); }

// Combined interface for convenience
interface ReadWriteFile extends Readable, Writable, Closeable {
    // Inherits all methods from parent interfaces
}

// Use the combined interface
class TextFile implements ReadWriteFile {
    @Override public String read() { /* ... */ }
    @Override public void write(String data) { /* ... */ }
    @Override public void close() { /* ... */ }
}

// Or use individual interfaces
class ReadOnlyFile implements Readable, Closeable {
    @Override public String read() { /* ... */ }
    @Override public void close() { /* ... */ }
}
```

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Principle** | Clients shouldn't depend on interfaces they don't use |
| **Problem** | Fat interfaces force unused dependencies |
| **Solution** | Create small, focused interfaces |
| **Benefit** | Flexible, decoupled, easier to implement |

### Key Questions:
- Does every client use EVERY method in this interface?
- Are there implementations with empty methods?
- Can this interface be split into logical groups?

### Remember:
- Many small interfaces > One fat interface
- Interfaces should be focused on ONE capability
- Clients should only know about what they actually use
- It's okay to have a class implement multiple interfaces

---

**Next: Dependency Inversion Principle (DIP) →**
