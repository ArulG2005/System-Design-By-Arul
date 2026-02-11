# Proxy Design Pattern

## Intent

> **Provide a surrogate or placeholder for another object to control access to it.**

---

## The Problem

You need to control access to an object:
- **Heavy object** - expensive to create (lazy loading)
- **Remote object** - exists on another server
- **Sensitive object** - needs access control
- **Logging/Caching** - add functionality without modifying original

---

## Simple Analogy

Think of a **Credit Card**:
- Credit card is a **proxy** for your bank account
- You don't carry cash (real object)
- Card controls access to your money
- Adds security (PIN verification)

Or think of a **Security Guard**:
- Guard is a **proxy** for a building
- Controls who can enter
- Checks credentials before allowing access

---

## Types of Proxy

```
┌────────────────────────────────────────────────────────────────┐
│                        PROXY TYPES                             │
├─────────────────┬──────────────────────────────────────────────┤
│ Virtual Proxy   │ Lazy loading of heavy objects                │
├─────────────────┼──────────────────────────────────────────────┤
│ Remote Proxy    │ Local representative for remote object       │
├─────────────────┼──────────────────────────────────────────────┤
│ Protection      │ Access control based on permissions          │
│ Proxy           │                                              │
├─────────────────┼──────────────────────────────────────────────┤
│ Logging Proxy   │ Logs all operations on real object          │
├─────────────────┼──────────────────────────────────────────────┤
│ Caching Proxy   │ Caches results of expensive operations       │
└─────────────────┴──────────────────────────────────────────────┘
```

---

## Structure

```
┌─────────────────────────────────────┐
│            Subject                  │
├─────────────────────────────────────┤
│ + request()                         │
└────────────────△────────────────────┘
                 │
        ┌────────┴────────┐
        │                 │
┌───────────────┐   ┌──────────────────────────┐
│  RealSubject  │   │         Proxy            │
├───────────────┤   ├──────────────────────────┤
│ + request()   │◄──│ - realSubject            │
└───────────────┘   │ + request()              │
                    │ - checkAccess()          │
                    │ - logAccess()            │
                    └──────────────────────────┘
                    
Client ──→ Proxy ──→ RealSubject
```

---

## Type 1: Virtual Proxy (Lazy Loading)

```java
// Subject interface
interface Image {
    void display();
}

// RealSubject - Heavy object
class RealImage implements Image {
    private String filename;
    
    public RealImage(String filename) {
        this.filename = filename;
        loadFromDisk();  // Expensive operation!
    }
    
    private void loadFromDisk() {
        System.out.println("Loading image: " + filename + " (takes 3 seconds...)");
        // Simulating heavy operation
        try { Thread.sleep(3000); } catch (Exception e) {}
        System.out.println("Image loaded: " + filename);
    }
    
    @Override
    public void display() {
        System.out.println("Displaying: " + filename);
    }
}

// Proxy - Lazy loading
class ImageProxy implements Image {
    private String filename;
    private RealImage realImage;  // Loaded only when needed
    
    public ImageProxy(String filename) {
        this.filename = filename;
        // Notice: NOT loading the image yet!
    }
    
    @Override
    public void display() {
        // Lazy loading - create only when first needed
        if (realImage == null) {
            realImage = new RealImage(filename);
        }
        realImage.display();
    }
}

// Usage
public class Gallery {
    public static void main(String[] args) {
        // Without proxy - all images load immediately (slow!)
        System.out.println("=== Without Proxy ===");
        Image img1 = new RealImage("photo1.jpg");  // 3 sec
        Image img2 = new RealImage("photo2.jpg");  // 3 sec
        Image img3 = new RealImage("photo3.jpg");  // 3 sec
        // Total: 9 seconds before app starts!
        
        System.out.println("\n=== With Proxy ===");
        // With proxy - instant creation
        Image proxy1 = new ImageProxy("photo1.jpg");  // Instant
        Image proxy2 = new ImageProxy("photo2.jpg");  // Instant
        Image proxy3 = new ImageProxy("photo3.jpg");  // Instant
        // App starts immediately!
        
        // Load only when needed
        System.out.println("\n--- User clicks on photo1 ---");
        proxy1.display();  // NOW it loads
        
        System.out.println("\n--- User clicks on photo1 again ---");
        proxy1.display();  // Already loaded, instant!
    }
}
```

---

## Type 2: Protection Proxy (Access Control)

```java
// Subject
interface Document {
    void display();
    void edit(String content);
    void delete();
}

// RealSubject
class RealDocument implements Document {
    private String name;
    private String content;
    
    public RealDocument(String name, String content) {
        this.name = name;
        this.content = content;
    }
    
    @Override
    public void display() {
        System.out.println("Document: " + name);
        System.out.println("Content: " + content);
    }
    
    @Override
    public void edit(String content) {
        this.content = content;
        System.out.println("Document edited successfully");
    }
    
    @Override
    public void delete() {
        System.out.println("Document deleted: " + name);
    }
}

// User class
class User {
    private String name;
    private String role;  // VIEWER, EDITOR, ADMIN
    
    public User(String name, String role) {
        this.name = name;
        this.role = role;
    }
    
    public String getName() { return name; }
    public String getRole() { return role; }
}

// Protection Proxy
class ProtectedDocument implements Document {
    private Document realDocument;
    private User user;
    
    public ProtectedDocument(Document document, User user) {
        this.realDocument = document;
        this.user = user;
    }
    
    @Override
    public void display() {
        // Everyone can view
        System.out.println("[" + user.getName() + "] accessing document...");
        realDocument.display();
    }
    
    @Override
    public void edit(String content) {
        // Only EDITOR and ADMIN can edit
        if (user.getRole().equals("EDITOR") || user.getRole().equals("ADMIN")) {
            System.out.println("[" + user.getName() + "] editing document...");
            realDocument.edit(content);
        } else {
            System.out.println("ACCESS DENIED: " + user.getName() + 
                             " (" + user.getRole() + ") cannot edit");
        }
    }
    
    @Override
    public void delete() {
        // Only ADMIN can delete
        if (user.getRole().equals("ADMIN")) {
            System.out.println("[" + user.getName() + "] deleting document...");
            realDocument.delete();
        } else {
            System.out.println("ACCESS DENIED: " + user.getName() + 
                             " (" + user.getRole() + ") cannot delete");
        }
    }
}

// Usage
public class DocumentSystem {
    public static void main(String[] args) {
        Document doc = new RealDocument("Report.docx", "Quarterly financials...");
        
        // Different users with different permissions
        User viewer = new User("Alice", "VIEWER");
        User editor = new User("Bob", "EDITOR");
        User admin = new User("Charlie", "ADMIN");
        
        System.out.println("=== VIEWER trying operations ===");
        Document viewerDoc = new ProtectedDocument(doc, viewer);
        viewerDoc.display();  // OK
        viewerDoc.edit("Hacked!");  // DENIED
        viewerDoc.delete();  // DENIED
        
        System.out.println("\n=== EDITOR trying operations ===");
        Document editorDoc = new ProtectedDocument(doc, editor);
        editorDoc.display();  // OK
        editorDoc.edit("Updated content");  // OK
        editorDoc.delete();  // DENIED
        
        System.out.println("\n=== ADMIN trying operations ===");
        Document adminDoc = new ProtectedDocument(doc, admin);
        adminDoc.display();  // OK
        adminDoc.edit("Admin changes");  // OK
        adminDoc.delete();  // OK
    }
}
```

---

## Type 3: Logging Proxy

```java
// Subject
interface Database {
    void insert(String data);
    String select(int id);
    void update(int id, String data);
    void delete(int id);
}

// RealSubject
class RealDatabase implements Database {
    private Map<Integer, String> storage = new HashMap<>();
    private int nextId = 1;
    
    @Override
    public void insert(String data) {
        storage.put(nextId++, data);
    }
    
    @Override
    public String select(int id) {
        return storage.get(id);
    }
    
    @Override
    public void update(int id, String data) {
        storage.put(id, data);
    }
    
    @Override
    public void delete(int id) {
        storage.remove(id);
    }
}

// Logging Proxy
class LoggingDatabaseProxy implements Database {
    private Database realDatabase;
    private List<String> logs = new ArrayList<>();
    
    public LoggingDatabaseProxy(Database database) {
        this.realDatabase = database;
    }
    
    private void log(String operation) {
        String timestamp = java.time.LocalDateTime.now().toString();
        String logEntry = "[" + timestamp + "] " + operation;
        logs.add(logEntry);
        System.out.println("LOG: " + logEntry);
    }
    
    @Override
    public void insert(String data) {
        log("INSERT: " + data);
        realDatabase.insert(data);
    }
    
    @Override
    public String select(int id) {
        log("SELECT: id=" + id);
        return realDatabase.select(id);
    }
    
    @Override
    public void update(int id, String data) {
        log("UPDATE: id=" + id + ", data=" + data);
        realDatabase.update(id, data);
    }
    
    @Override
    public void delete(int id) {
        log("DELETE: id=" + id);
        realDatabase.delete(id);
    }
    
    public List<String> getLogs() {
        return new ArrayList<>(logs);
    }
}

// Usage
public class App {
    public static void main(String[] args) {
        Database db = new LoggingDatabaseProxy(new RealDatabase());
        
        db.insert("User: John");
        db.insert("User: Jane");
        db.select(1);
        db.update(1, "User: John Doe");
        db.delete(2);
        
        // All operations are logged!
    }
}
```

---

## Type 4: Caching Proxy

```java
// Subject
interface YouTubeService {
    String getVideo(String videoId);
    List<String> getVideoList(String channel);
}

// RealSubject - Slow network calls
class RealYouTubeService implements YouTubeService {
    @Override
    public String getVideo(String videoId) {
        System.out.println("  Downloading video " + videoId + " from YouTube...");
        // Simulating slow download
        sleep(2000);
        return "Video content: " + videoId;
    }
    
    @Override
    public List<String> getVideoList(String channel) {
        System.out.println("  Fetching video list for " + channel + "...");
        sleep(1000);
        return Arrays.asList("video1", "video2", "video3");
    }
    
    private void sleep(int ms) {
        try { Thread.sleep(ms); } catch (Exception e) {}
    }
}

// Caching Proxy
class CachedYouTubeProxy implements YouTubeService {
    private YouTubeService realService;
    private Map<String, String> videoCache = new HashMap<>();
    private Map<String, List<String>> listCache = new HashMap<>();
    
    public CachedYouTubeProxy() {
        this.realService = new RealYouTubeService();
    }
    
    @Override
    public String getVideo(String videoId) {
        if (!videoCache.containsKey(videoId)) {
            System.out.println("Cache MISS - fetching from server");
            videoCache.put(videoId, realService.getVideo(videoId));
        } else {
            System.out.println("Cache HIT - returning cached video");
        }
        return videoCache.get(videoId);
    }
    
    @Override
    public List<String> getVideoList(String channel) {
        if (!listCache.containsKey(channel)) {
            System.out.println("Cache MISS - fetching from server");
            listCache.put(channel, realService.getVideoList(channel));
        } else {
            System.out.println("Cache HIT - returning cached list");
        }
        return listCache.get(channel);
    }
    
    public void clearCache() {
        videoCache.clear();
        listCache.clear();
        System.out.println("Cache cleared!");
    }
}

// Usage
public class VideoApp {
    public static void main(String[] args) {
        YouTubeService youtube = new CachedYouTubeProxy();
        
        System.out.println("=== First request (slow) ===");
        youtube.getVideo("abc123");  // Downloads
        
        System.out.println("\n=== Second request (fast) ===");
        youtube.getVideo("abc123");  // From cache!
        
        System.out.println("\n=== Different video (slow) ===");
        youtube.getVideo("xyz789");  // Downloads
        
        System.out.println("\n=== First video again (fast) ===");
        youtube.getVideo("abc123");  // From cache!
    }
}
```

---

## Type 5: Remote Proxy (Conceptual)

```java
// Subject
interface RemoteService {
    String getData();
    void setData(String data);
}

// Remote Proxy - represents remote object
class RemoteServiceProxy implements RemoteService {
    private String serverUrl;
    
    public RemoteServiceProxy(String serverUrl) {
        this.serverUrl = serverUrl;
    }
    
    @Override
    public String getData() {
        System.out.println("Connecting to " + serverUrl + "...");
        System.out.println("Sending GET request...");
        // In reality: make HTTP call, deserialize response
        return "Data from remote server";
    }
    
    @Override
    public void setData(String data) {
        System.out.println("Connecting to " + serverUrl + "...");
        System.out.println("Sending POST request with: " + data);
        // In reality: make HTTP call
    }
}

// Client uses proxy like local object
public class Client {
    public static void main(String[] args) {
        // Looks like local object!
        RemoteService service = new RemoteServiceProxy("http://api.server.com");
        
        // But actually talks to remote server
        String data = service.getData();
        service.setData("Updated value");
    }
}
```

---

## Real-World Example: Smart Reference Proxy

```java
// Subject
interface HeavyResource {
    void process();
}

// Real heavy resource
class RealHeavyResource implements HeavyResource {
    public RealHeavyResource() {
        System.out.println("Creating expensive resource...");
    }
    
    @Override
    public void process() {
        System.out.println("Processing with heavy resource");
    }
}

// Smart Reference Proxy with reference counting
class SmartResourceProxy implements HeavyResource {
    private static RealHeavyResource sharedResource;
    private static int referenceCount = 0;
    
    public SmartResourceProxy() {
        referenceCount++;
        System.out.println("New reference. Count: " + referenceCount);
        
        if (sharedResource == null) {
            sharedResource = new RealHeavyResource();
        }
    }
    
    @Override
    public void process() {
        sharedResource.process();
    }
    
    public void release() {
        referenceCount--;
        System.out.println("Released. Count: " + referenceCount);
        
        if (referenceCount == 0) {
            System.out.println("No more references. Cleaning up resource.");
            sharedResource = null;
        }
    }
}

// Usage
public class ResourceManager {
    public static void main(String[] args) {
        SmartResourceProxy proxy1 = new SmartResourceProxy();  // Creates resource
        SmartResourceProxy proxy2 = new SmartResourceProxy();  // Reuses
        SmartResourceProxy proxy3 = new SmartResourceProxy();  // Reuses
        
        proxy1.process();
        proxy2.process();
        
        proxy1.release();
        proxy2.release();
        proxy3.release();  // Resource cleaned up
    }
}
```

---

## When to Use Proxy Pattern

### ✅ Use When:
1. **Lazy loading** - delay expensive object creation
2. **Access control** - check permissions
3. **Logging/Monitoring** - track usage
4. **Caching** - avoid repeated expensive operations
5. **Remote access** - local interface for remote objects

### ❌ Don't Use When:
1. Direct access is fast and simple
2. No need for additional control
3. Would add unnecessary complexity

---

## Proxy vs Other Patterns

| Pattern | Purpose |
|---------|---------|
| **Proxy** | Control access to object |
| **Decorator** | Add behavior dynamically |
| **Adapter** | Convert interface |
| **Facade** | Simplify complex subsystem |

### Key Difference from Decorator:
- **Decorator** adds behavior
- **Proxy** controls access (same behavior)

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Intent** | Control access to another object |
| **Types** | Virtual, Protection, Remote, Logging, Caching |
| **Benefits** | Lazy loading, access control, caching, logging |
| **Structure** | Proxy → RealSubject (same interface) |

### Remember:
- Proxy and RealSubject share **same interface**
- Client doesn't know if it's using proxy or real
- Proxy **controls** access, doesn't extend functionality

---

**Next: Bridge Pattern →**
