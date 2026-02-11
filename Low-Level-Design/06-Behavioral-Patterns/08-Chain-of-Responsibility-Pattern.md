# Chain of Responsibility Pattern

## Intent

> **Avoid coupling the sender of a request to its receiver by giving more than one object a chance to handle the request. Chain the receiving objects and pass the request along the chain until an object handles it.**

---

## The Problem

You need to **process a request** but:
- Multiple handlers could potentially handle it
- You don't want sender to know which handler processes it
- Handler should be **determined at runtime**
- You want to **decouple** request sender from handlers

### Bad Approach: Hard-coded checks

```java
// Tightly coupled - BAD! ❌
class SupportTicket {
    void process(Request request) {
        if (request.getLevel() == 1) {
            basicSupport.handle(request);
        } else if (request.getLevel() == 2) {
            technicalSupport.handle(request);
        } else if (request.getLevel() == 3) {
            managerSupport.handle(request);
        } else if (request.getLevel() == 4) {
            directorSupport.handle(request);
        }
        // Adding new level = modifying this method
    }
}
```

---

## Simple Analogy

Think of a **Customer Support System**:
1. You call customer service
2. **Level 1 agent** tries to help → Can't solve? → Transfer
3. **Level 2 technical** tries to help → Can't solve? → Transfer
4. **Manager** handles escalated issue

Or think of **Exception Handling**:
```java
try {
    // code
} catch (IOException e) {
    // Handler 1
} catch (SQLException e) {
    // Handler 2
} catch (Exception e) {
    // Handler 3 (fallback)
}
```

---

## Structure

```
┌─────────────────────────────────────────┐
│              Client                      │
│  handler.handle(request)                 │
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                       «abstract» Handler                                   │
├───────────────────────────────────────────────────────────────────────────┤
│  - nextHandler: Handler                                                    │
├───────────────────────────────────────────────────────────────────────────┤
│  + setNext(Handler): Handler                                              │
│  + handle(Request)                                                         │
└────────────────────────────────────────────────────────────────△──────────┘
                                                                  │
               ┌──────────────────────────────────────────────────┼──────────┐
               │                                                  │          │
   ┌───────────────────────┐            ┌───────────────────────┐    ┌───────────────┐
   │   ConcreteHandler1    │───next───> │   ConcreteHandler2    │──> │ ConcreteHandler3│
   ├───────────────────────┤            ├───────────────────────┤    ├───────────────┤
   │ + handle(Request)     │            │ + handle(Request)     │    │ + handle()    │
   │   if can handle:      │            │   if can handle:      │    │               │
   │     process           │            │     process           │    │               │
   │   else:               │            │   else:               │    │               │
   │     next.handle()     │            │     next.handle()     │    │               │
   └───────────────────────┘            └───────────────────────┘    └───────────────┘
```

---

## Basic Example: Logging

```java
// Request
class LogMessage {
    public enum Level { DEBUG, INFO, WARNING, ERROR }
    
    private Level level;
    private String message;
    
    public LogMessage(Level level, String message) {
        this.level = level;
        this.message = message;
    }
    
    public Level getLevel() { return level; }
    public String getMessage() { return message; }
}

// Handler interface
abstract class Logger {
    protected Logger nextLogger;
    
    public Logger setNext(Logger next) {
        this.nextLogger = next;
        return next;  // For chaining
    }
    
    public void log(LogMessage message) {
        if (canHandle(message)) {
            write(message);
        }
        // Pass to next handler (all loggers process based on level)
        if (nextLogger != null) {
            nextLogger.log(message);
        }
    }
    
    protected abstract boolean canHandle(LogMessage message);
    protected abstract void write(LogMessage message);
}

// Concrete Handlers
class ConsoleLogger extends Logger {
    @Override
    protected boolean canHandle(LogMessage message) {
        // Console logs everything
        return true;
    }
    
    @Override
    protected void write(LogMessage message) {
        System.out.println("[CONSOLE] " + message.getLevel() + ": " + message.getMessage());
    }
}

class FileLogger extends Logger {
    @Override
    protected boolean canHandle(LogMessage message) {
        // File logs INFO and above
        return message.getLevel().ordinal() >= LogMessage.Level.INFO.ordinal();
    }
    
    @Override
    protected void write(LogMessage message) {
        System.out.println("[FILE] " + message.getLevel() + ": " + message.getMessage());
    }
}

class ErrorLogger extends Logger {
    @Override
    protected boolean canHandle(LogMessage message) {
        // Error logger only for ERROR
        return message.getLevel() == LogMessage.Level.ERROR;
    }
    
    @Override
    protected void write(LogMessage message) {
        System.out.println("[ERROR ALERT] 🚨 " + message.getMessage());
    }
}

// Usage
public class LoggingDemo {
    public static void main(String[] args) {
        // Build chain
        Logger console = new ConsoleLogger();
        Logger file = new FileLogger();
        Logger error = new ErrorLogger();
        
        console.setNext(file).setNext(error);
        
        // Test
        console.log(new LogMessage(LogMessage.Level.DEBUG, "Debug info"));
        System.out.println();
        
        console.log(new LogMessage(LogMessage.Level.INFO, "User logged in"));
        System.out.println();
        
        console.log(new LogMessage(LogMessage.Level.ERROR, "Database connection failed!"));
    }
}

// Output:
// [CONSOLE] DEBUG: Debug info
// 
// [CONSOLE] INFO: User logged in
// [FILE] INFO: User logged in
// 
// [CONSOLE] ERROR: Database connection failed!
// [FILE] ERROR: Database connection failed!
// [ERROR ALERT] 🚨 Database connection failed!
```

---

## Real-World Examples

### Example 1: Customer Support System

```java
// Request
class SupportTicket {
    public enum Priority { LOW, MEDIUM, HIGH, CRITICAL }
    
    private String issue;
    private Priority priority;
    private String customerType;  // "regular", "premium", "vip"
    
    public SupportTicket(String issue, Priority priority, String customerType) {
        this.issue = issue;
        this.priority = priority;
        this.customerType = customerType;
    }
    
    public String getIssue() { return issue; }
    public Priority getPriority() { return priority; }
    public String getCustomerType() { return customerType; }
}

// Handler
abstract class SupportHandler {
    protected SupportHandler nextHandler;
    protected String handlerName;
    
    public SupportHandler(String name) {
        this.handlerName = name;
    }
    
    public SupportHandler setNext(SupportHandler next) {
        this.nextHandler = next;
        return next;
    }
    
    public void handle(SupportTicket ticket) {
        if (canHandle(ticket)) {
            process(ticket);
        } else if (nextHandler != null) {
            System.out.println(handlerName + " → Escalating to next level");
            nextHandler.handle(ticket);
        } else {
            System.out.println("No handler available for: " + ticket.getIssue());
        }
    }
    
    protected abstract boolean canHandle(SupportTicket ticket);
    
    protected void process(SupportTicket ticket) {
        System.out.println("✓ " + handlerName + " handling: " + ticket.getIssue());
    }
}

// Concrete Handlers
class FrontDesk extends SupportHandler {
    public FrontDesk() {
        super("Front Desk");
    }
    
    @Override
    protected boolean canHandle(SupportTicket ticket) {
        // Handle low priority regular customers
        return ticket.getPriority() == SupportTicket.Priority.LOW 
            && ticket.getCustomerType().equals("regular");
    }
}

class TechnicalSupport extends SupportHandler {
    public TechnicalSupport() {
        super("Technical Support");
    }
    
    @Override
    protected boolean canHandle(SupportTicket ticket) {
        // Handle low-medium priority
        return ticket.getPriority().ordinal() <= SupportTicket.Priority.MEDIUM.ordinal();
    }
}

class SeniorTechnical extends SupportHandler {
    public SeniorTechnical() {
        super("Senior Technical");
    }
    
    @Override
    protected boolean canHandle(SupportTicket ticket) {
        // Handle up to high priority
        return ticket.getPriority().ordinal() <= SupportTicket.Priority.HIGH.ordinal();
    }
}

class Manager extends SupportHandler {
    public Manager() {
        super("Manager");
    }
    
    @Override
    protected boolean canHandle(SupportTicket ticket) {
        // Handle critical or VIP customers
        return ticket.getPriority() == SupportTicket.Priority.CRITICAL
            || ticket.getCustomerType().equals("vip");
    }
}

class Director extends SupportHandler {
    public Director() {
        super("Director");
    }
    
    @Override
    protected boolean canHandle(SupportTicket ticket) {
        // Ultimate fallback - handles everything
        return true;
    }
}

// Usage
public class SupportDemo {
    public static void main(String[] args) {
        // Build chain
        SupportHandler frontDesk = new FrontDesk();
        SupportHandler tech = new TechnicalSupport();
        SupportHandler seniorTech = new SeniorTechnical();
        SupportHandler manager = new Manager();
        SupportHandler director = new Director();
        
        frontDesk.setNext(tech).setNext(seniorTech).setNext(manager).setNext(director);
        
        System.out.println("=== Ticket 1: Low priority regular ===");
        frontDesk.handle(new SupportTicket("Password reset", 
            SupportTicket.Priority.LOW, "regular"));
        
        System.out.println("\n=== Ticket 2: Medium priority ===");
        frontDesk.handle(new SupportTicket("Software bug", 
            SupportTicket.Priority.MEDIUM, "premium"));
        
        System.out.println("\n=== Ticket 3: Critical VIP ===");
        frontDesk.handle(new SupportTicket("System down", 
            SupportTicket.Priority.CRITICAL, "vip"));
    }
}
```

---

### Example 2: HTTP Middleware Pipeline

```java
// Request/Response objects
class HttpRequest {
    private Map<String, String> headers = new HashMap<>();
    private String body;
    private String path;
    private String method;
    private String userId;
    private boolean authenticated = false;
    
    public HttpRequest(String method, String path) {
        this.method = method;
        this.path = path;
    }
    
    // Getters and setters
    public void setHeader(String key, String value) { headers.put(key, value); }
    public String getHeader(String key) { return headers.get(key); }
    public String getPath() { return path; }
    public String getMethod() { return method; }
    public void setAuthenticated(boolean auth) { authenticated = auth; }
    public boolean isAuthenticated() { return authenticated; }
    public void setUserId(String userId) { this.userId = userId; }
    public String getUserId() { return userId; }
}

class HttpResponse {
    private int statusCode = 200;
    private String body = "";
    
    public void setStatus(int code) { statusCode = code; }
    public void setBody(String body) { this.body = body; }
    public int getStatusCode() { return statusCode; }
    public String getBody() { return body; }
}

// Middleware interface
abstract class Middleware {
    protected Middleware next;
    
    public Middleware linkWith(Middleware next) {
        this.next = next;
        return next;
    }
    
    public abstract boolean handle(HttpRequest request, HttpResponse response);
    
    protected boolean handleNext(HttpRequest request, HttpResponse response) {
        if (next == null) {
            return true;
        }
        return next.handle(request, response);
    }
}

// Concrete Middleware

// 1. Logging Middleware
class LoggingMiddleware extends Middleware {
    @Override
    public boolean handle(HttpRequest request, HttpResponse response) {
        System.out.println("📝 [LOG] " + request.getMethod() + " " + request.getPath());
        boolean result = handleNext(request, response);
        System.out.println("📝 [LOG] Response: " + response.getStatusCode());
        return result;
    }
}

// 2. Authentication Middleware
class AuthMiddleware extends Middleware {
    @Override
    public boolean handle(HttpRequest request, HttpResponse response) {
        String token = request.getHeader("Authorization");
        
        if (token == null || token.isEmpty()) {
            response.setStatus(401);
            response.setBody("Unauthorized: No token provided");
            System.out.println("🔐 [AUTH] Rejected - No token");
            return false;
        }
        
        // Validate token (simplified)
        if (token.startsWith("Bearer ")) {
            request.setAuthenticated(true);
            request.setUserId("user123");
            System.out.println("🔐 [AUTH] Authenticated user: " + request.getUserId());
            return handleNext(request, response);
        }
        
        response.setStatus(401);
        response.setBody("Unauthorized: Invalid token");
        System.out.println("🔐 [AUTH] Rejected - Invalid token");
        return false;
    }
}

// 3. Rate Limiting Middleware
class RateLimitMiddleware extends Middleware {
    private Map<String, Integer> requestCounts = new HashMap<>();
    private int limit = 10;
    
    @Override
    public boolean handle(HttpRequest request, HttpResponse response) {
        String userId = request.getUserId();
        if (userId == null) userId = "anonymous";
        
        int count = requestCounts.getOrDefault(userId, 0) + 1;
        requestCounts.put(userId, count);
        
        if (count > limit) {
            response.setStatus(429);
            response.setBody("Too Many Requests");
            System.out.println("⏱️ [RATE] Limit exceeded for: " + userId);
            return false;
        }
        
        System.out.println("⏱️ [RATE] Request " + count + "/" + limit + " for: " + userId);
        return handleNext(request, response);
    }
}

// 4. Validation Middleware
class ValidationMiddleware extends Middleware {
    @Override
    public boolean handle(HttpRequest request, HttpResponse response) {
        if (request.getMethod().equals("POST") || request.getMethod().equals("PUT")) {
            // Check content type
            String contentType = request.getHeader("Content-Type");
            if (contentType == null || !contentType.contains("application/json")) {
                response.setStatus(400);
                response.setBody("Bad Request: Content-Type must be application/json");
                System.out.println("✅ [VALID] Rejected - Invalid content type");
                return false;
            }
        }
        System.out.println("✅ [VALID] Request validated");
        return handleNext(request, response);
    }
}

// 5. Controller/Handler (final handler)
class ControllerMiddleware extends Middleware {
    @Override
    public boolean handle(HttpRequest request, HttpResponse response) {
        System.out.println("🎯 [CONTROLLER] Processing: " + request.getPath());
        
        // Route handling
        switch (request.getPath()) {
            case "/api/users":
                response.setBody("{\"users\": [\"John\", \"Jane\"]}");
                break;
            case "/api/products":
                response.setBody("{\"products\": [\"Laptop\", \"Phone\"]}");
                break;
            default:
                response.setStatus(404);
                response.setBody("Not Found");
        }
        
        return true;
    }
}

// Usage
public class MiddlewareDemo {
    public static void main(String[] args) {
        // Build middleware chain
        Middleware logging = new LoggingMiddleware();
        Middleware auth = new AuthMiddleware();
        Middleware rateLimit = new RateLimitMiddleware();
        Middleware validation = new ValidationMiddleware();
        Middleware controller = new ControllerMiddleware();
        
        logging.linkWith(auth).linkWith(rateLimit).linkWith(validation).linkWith(controller);
        
        // Test 1: Valid request
        System.out.println("=== Test 1: Valid Request ===");
        HttpRequest req1 = new HttpRequest("GET", "/api/users");
        req1.setHeader("Authorization", "Bearer valid-token");
        HttpResponse res1 = new HttpResponse();
        logging.handle(req1, res1);
        System.out.println("Response Body: " + res1.getBody() + "\n");
        
        // Test 2: No auth token
        System.out.println("=== Test 2: No Auth Token ===");
        HttpRequest req2 = new HttpRequest("GET", "/api/users");
        HttpResponse res2 = new HttpResponse();
        logging.handle(req2, res2);
        System.out.println("Response: " + res2.getStatusCode() + " - " + res2.getBody() + "\n");
    }
}
```

---

### Example 3: Purchase Approval

```java
// Request
class PurchaseRequest {
    private String purpose;
    private double amount;
    private String department;
    
    public PurchaseRequest(String purpose, double amount, String department) {
        this.purpose = purpose;
        this.amount = amount;
        this.department = department;
    }
    
    public String getPurpose() { return purpose; }
    public double getAmount() { return amount; }
    public String getDepartment() { return department; }
    
    @Override
    public String toString() {
        return String.format("%s ($%.2f) - %s", purpose, amount, department);
    }
}

// Handler
abstract class Approver {
    protected Approver nextApprover;
    protected String title;
    protected double approvalLimit;
    
    public Approver(String title, double limit) {
        this.title = title;
        this.approvalLimit = limit;
    }
    
    public Approver setNext(Approver next) {
        this.nextApprover = next;
        return next;
    }
    
    public void processRequest(PurchaseRequest request) {
        if (request.getAmount() <= approvalLimit) {
            approve(request);
        } else if (nextApprover != null) {
            System.out.println(title + ": Amount $" + request.getAmount() 
                + " exceeds my limit $" + approvalLimit + ". Escalating...");
            nextApprover.processRequest(request);
        } else {
            deny(request);
        }
    }
    
    protected void approve(PurchaseRequest request) {
        System.out.println("✓ " + title + " APPROVED: " + request);
    }
    
    protected void deny(PurchaseRequest request) {
        System.out.println("✗ DENIED: " + request + " - Exceeds all approval limits");
    }
}

// Concrete Approvers
class TeamLead extends Approver {
    public TeamLead() {
        super("Team Lead", 1000);
    }
}

class DepartmentHead extends Approver {
    public DepartmentHead() {
        super("Department Head", 5000);
    }
}

class VicePresident extends Approver {
    public VicePresident() {
        super("Vice President", 25000);
    }
}

class CEO extends Approver {
    public CEO() {
        super("CEO", 100000);
    }
}

class Board extends Approver {
    public Board() {
        super("Board of Directors", Double.MAX_VALUE);
    }
}

// Usage
public class ApprovalDemo {
    public static void main(String[] args) {
        // Build chain
        Approver teamLead = new TeamLead();
        Approver deptHead = new DepartmentHead();
        Approver vp = new VicePresident();
        Approver ceo = new CEO();
        Approver board = new Board();
        
        teamLead.setNext(deptHead).setNext(vp).setNext(ceo).setNext(board);
        
        // Test requests
        PurchaseRequest[] requests = {
            new PurchaseRequest("Office supplies", 500, "Admin"),
            new PurchaseRequest("Software license", 3500, "IT"),
            new PurchaseRequest("Server upgrade", 15000, "IT"),
            new PurchaseRequest("New department setup", 75000, "Expansion"),
            new PurchaseRequest("Acquire competitor", 500000, "M&A")
        };
        
        for (PurchaseRequest req : requests) {
            System.out.println("\n--- Processing: " + req + " ---");
            teamLead.processRequest(req);
        }
    }
}
```

---

## Variations

### 1. All Handlers Process (Logging example)
```java
// Each handler processes AND passes to next
public void handle(Request req) {
    process(req);  // Always process
    if (next != null) {
        next.handle(req);  // Always pass
    }
}
```

### 2. First Match Handles (Approval example)
```java
// First handler that can handle, stops the chain
public void handle(Request req) {
    if (canHandle(req)) {
        process(req);  // Handle and stop
    } else if (next != null) {
        next.handle(req);  // Pass to next
    }
}
```

### 3. Transform and Pass
```java
// Each handler transforms request
public Request handle(Request req) {
    Request transformed = transform(req);
    if (next != null) {
        return next.handle(transformed);
    }
    return transformed;
}
```

---

## When to Use

### ✅ Use When:
1. **Multiple handlers** could process a request
2. Handler set should be **dynamic** or configurable
3. You want to **decouple sender** from receivers
4. Processing should happen in **order**

### ❌ Don't Use When:
1. There's always only **one handler**
2. Request must be handled (**guaranteed** handler exists)
3. Chain would be too **long and slow**

---

## Chain of Responsibility vs Other Patterns

| Pattern | Purpose |
|---------|---------|
| **Chain of Responsibility** | Pass request along chain until handled |
| **Command** | Encapsulate request as object |
| **Decorator** | Add behavior by wrapping |
| **Composite** | Treat group as individual |

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Intent** | Give multiple objects chance to handle request |
| **Key Idea** | Chain handlers, pass request until handled |
| **Benefits** | Decoupling, flexible handler order, SRP |
| **Use Cases** | Middleware, logging, approval workflows, event handling |

### Remember:
- **Sender** doesn't know which handler processes
- Handlers form a **chain** (linked list)
- Request passes until **handled** or reaches **end**
- Great for **middleware** pipelines!

---

**Next: Mediator Pattern →**
