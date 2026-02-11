# Exception Handling and Building Resilient Systems

## Why Proper Error Handling Matters

Bad error handling leads to:
- **System crashes** from unhandled exceptions
- **Data corruption** from partial operations
- **Poor user experience** with cryptic error messages
- **Difficult debugging** without proper logging
- **Security vulnerabilities** from exposed stack traces

---

## Exception Handling Best Practices

### 1. Use Specific Exception Types

```java
// BAD: Catching everything ❌
try {
    processOrder(order);
} catch (Exception e) {
    System.out.println("Something went wrong");  // What went wrong?!
}

// GOOD: Specific exception handling ✅
try {
    processOrder(order);
} catch (InsufficientStockException e) {
    notifyUser("Sorry, item is out of stock");
    suggestAlternatives(e.getProductId());
} catch (PaymentDeclinedException e) {
    notifyUser("Payment failed: " + e.getReason());
    offerAlternativePayment();
} catch (DatabaseException e) {
    logger.error("Database error", e);
    notifyUser("Service temporarily unavailable. Please try again.");
}
```

---

### 2. Create Custom Exceptions

```java
// Base business exception
public abstract class BusinessException extends RuntimeException {
    private final String errorCode;
    private final Map<String, Object> context;
    
    public BusinessException(String message, String errorCode) {
        super(message);
        this.errorCode = errorCode;
        this.context = new HashMap<>();
    }
    
    public BusinessException addContext(String key, Object value) {
        context.put(key, value);
        return this;
    }
    
    public String getErrorCode() { return errorCode; }
    public Map<String, Object> getContext() { return context; }
}

// Specific exceptions
public class OrderNotFoundException extends BusinessException {
    public OrderNotFoundException(String orderId) {
        super("Order not found: " + orderId, "ORDER_NOT_FOUND");
        addContext("orderId", orderId);
    }
}

public class InsufficientFundsException extends BusinessException {
    public InsufficientFundsException(double required, double available) {
        super("Insufficient funds", "INSUFFICIENT_FUNDS");
        addContext("required", required);
        addContext("available", available);
    }
}

public class InvalidOperationException extends BusinessException {
    public InvalidOperationException(String operation, String reason) {
        super("Invalid operation: " + operation, "INVALID_OPERATION");
        addContext("operation", operation);
        addContext("reason", reason);
    }
}

// Usage
public class OrderService {
    public Order getOrder(String orderId) {
        Order order = repository.findById(orderId);
        if (order == null) {
            throw new OrderNotFoundException(orderId);
        }
        return order;
    }
    
    public void refundOrder(Order order) {
        if (order.getStatus() != OrderStatus.DELIVERED) {
            throw new InvalidOperationException(
                "refund", 
                "Can only refund delivered orders"
            );
        }
        // Process refund...
    }
}
```

---

### 3. Exception Hierarchy

```java
// Well-structured exception hierarchy
public abstract class ApplicationException extends RuntimeException {
    // Base for all application exceptions
}

// Business logic exceptions (client errors)
public abstract class BusinessException extends ApplicationException {
}

public class ValidationException extends BusinessException { }
public class ResourceNotFoundException extends BusinessException { }
public class AccessDeniedException extends BusinessException { }
public class DuplicateResourceException extends BusinessException { }

// Technical exceptions (server errors)
public abstract class TechnicalException extends ApplicationException {
}

public class DatabaseException extends TechnicalException { }
public class ExternalServiceException extends TechnicalException { }
public class ConfigurationException extends TechnicalException { }
```

---

### 4. Result Object Pattern (Alternative to Exceptions)

```java
// Result class - wraps success or failure
public class Result<T> {
    private final T value;
    private final String error;
    private final String errorCode;
    private final boolean success;
    
    private Result(T value, String error, String errorCode, boolean success) {
        this.value = value;
        this.error = error;
        this.errorCode = errorCode;
        this.success = success;
    }
    
    public static <T> Result<T> success(T value) {
        return new Result<>(value, null, null, true);
    }
    
    public static <T> Result<T> failure(String error, String errorCode) {
        return new Result<>(null, error, errorCode, false);
    }
    
    public boolean isSuccess() { return success; }
    public boolean isFailure() { return !success; }
    public T getValue() { return value; }
    public String getError() { return error; }
    public String getErrorCode() { return errorCode; }
    
    // Fluent methods
    public <U> Result<U> map(Function<T, U> mapper) {
        if (isSuccess()) {
            return Result.success(mapper.apply(value));
        }
        return Result.failure(error, errorCode);
    }
    
    public T orElse(T defaultValue) {
        return isSuccess() ? value : defaultValue;
    }
    
    public T orElseThrow(Supplier<? extends RuntimeException> exceptionSupplier) {
        if (isSuccess()) return value;
        throw exceptionSupplier.get();
    }
}

// Usage
public class PaymentService {
    public Result<PaymentConfirmation> processPayment(PaymentRequest request) {
        // Validation
        if (request.getAmount() <= 0) {
            return Result.failure("Invalid amount", "INVALID_AMOUNT");
        }
        
        // Check balance
        double balance = getBalance(request.getUserId());
        if (balance < request.getAmount()) {
            return Result.failure("Insufficient funds", "INSUFFICIENT_FUNDS");
        }
        
        // Process payment
        try {
            PaymentConfirmation confirmation = gateway.process(request);
            return Result.success(confirmation);
        } catch (GatewayException e) {
            return Result.failure("Payment gateway error", "GATEWAY_ERROR");
        }
    }
}

// Client code
Result<PaymentConfirmation> result = paymentService.processPayment(request);

if (result.isSuccess()) {
    showConfirmation(result.getValue());
} else {
    showError(result.getError());
}

// Or with fluent API
result.map(PaymentConfirmation::getTransactionId)
      .orElse("NO_TRANSACTION");
```

---

## Building Resilient Systems

### 1. Retry Pattern

```java
public class RetryExecutor {
    private final int maxAttempts;
    private final long delayMs;
    private final double backoffMultiplier;
    
    public RetryExecutor(int maxAttempts, long delayMs, double backoffMultiplier) {
        this.maxAttempts = maxAttempts;
        this.delayMs = delayMs;
        this.backoffMultiplier = backoffMultiplier;
    }
    
    public <T> T execute(Supplier<T> operation, Predicate<Exception> retryCondition) {
        Exception lastException = null;
        long currentDelay = delayMs;
        
        for (int attempt = 1; attempt <= maxAttempts; attempt++) {
            try {
                return operation.get();
            } catch (Exception e) {
                lastException = e;
                
                if (!retryCondition.test(e) || attempt == maxAttempts) {
                    break;
                }
                
                System.out.println("Attempt " + attempt + " failed. Retrying in " + 
                                  currentDelay + "ms...");
                
                try {
                    Thread.sleep(currentDelay);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    throw new RuntimeException("Retry interrupted", ie);
                }
                
                currentDelay = (long) (currentDelay * backoffMultiplier);
            }
        }
        
        throw new RuntimeException("All retry attempts failed", lastException);
    }
}

// Usage
RetryExecutor retry = new RetryExecutor(3, 1000, 2.0);  // 3 attempts, 1s initial delay, 2x backoff

String result = retry.execute(
    () -> externalService.fetchData(),
    e -> e instanceof TimeoutException || e instanceof ServiceUnavailableException
);
```

---

### 2. Circuit Breaker Pattern

```java
public class CircuitBreaker {
    private final int failureThreshold;
    private final long resetTimeoutMs;
    
    private int failureCount = 0;
    private long lastFailureTime = 0;
    private State state = State.CLOSED;
    
    public enum State {
        CLOSED,      // Normal operation
        OPEN,        // Failing fast
        HALF_OPEN    // Testing if service recovered
    }
    
    public CircuitBreaker(int failureThreshold, long resetTimeoutMs) {
        this.failureThreshold = failureThreshold;
        this.resetTimeoutMs = resetTimeoutMs;
    }
    
    public <T> T execute(Supplier<T> operation) {
        if (state == State.OPEN) {
            if (System.currentTimeMillis() - lastFailureTime > resetTimeoutMs) {
                state = State.HALF_OPEN;
                System.out.println("Circuit breaker: HALF_OPEN (testing)");
            } else {
                throw new CircuitBreakerOpenException("Circuit breaker is OPEN");
            }
        }
        
        try {
            T result = operation.get();
            onSuccess();
            return result;
        } catch (Exception e) {
            onFailure();
            throw e;
        }
    }
    
    private synchronized void onSuccess() {
        failureCount = 0;
        if (state == State.HALF_OPEN) {
            state = State.CLOSED;
            System.out.println("Circuit breaker: CLOSED (recovered)");
        }
    }
    
    private synchronized void onFailure() {
        failureCount++;
        lastFailureTime = System.currentTimeMillis();
        
        if (failureCount >= failureThreshold) {
            state = State.OPEN;
            System.out.println("Circuit breaker: OPEN (failing fast)");
        }
    }
    
    public State getState() { return state; }
}

class CircuitBreakerOpenException extends RuntimeException {
    public CircuitBreakerOpenException(String message) {
        super(message);
    }
}

// Usage
CircuitBreaker circuitBreaker = new CircuitBreaker(5, 30000);  // 5 failures, 30s reset

public Data fetchData() {
    try {
        return circuitBreaker.execute(() -> externalService.getData());
    } catch (CircuitBreakerOpenException e) {
        return getCachedData();  // Fallback
    }
}
```

---

### 3. Timeout Pattern

```java
public class TimeoutExecutor {
    private final ExecutorService executor = Executors.newCachedThreadPool();
    
    public <T> T executeWithTimeout(Callable<T> task, long timeoutMs) {
        Future<T> future = executor.submit(task);
        
        try {
            return future.get(timeoutMs, TimeUnit.MILLISECONDS);
        } catch (TimeoutException e) {
            future.cancel(true);
            throw new OperationTimeoutException("Operation timed out after " + timeoutMs + "ms");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("Operation interrupted", e);
        } catch (ExecutionException e) {
            throw new RuntimeException("Operation failed", e.getCause());
        }
    }
}

class OperationTimeoutException extends RuntimeException {
    public OperationTimeoutException(String message) {
        super(message);
    }
}

// Usage
TimeoutExecutor timeoutExecutor = new TimeoutExecutor();

Data data = timeoutExecutor.executeWithTimeout(
    () -> slowExternalService.fetchData(),
    5000  // 5 second timeout
);
```

---

### 4. Fallback Pattern

```java
public class FallbackService<T> {
    private final Supplier<T> primary;
    private final Supplier<T> fallback;
    private final Predicate<Exception> fallbackCondition;
    
    public FallbackService(Supplier<T> primary, Supplier<T> fallback, 
                          Predicate<Exception> fallbackCondition) {
        this.primary = primary;
        this.fallback = fallback;
        this.fallbackCondition = fallbackCondition;
    }
    
    public T execute() {
        try {
            return primary.get();
        } catch (Exception e) {
            if (fallbackCondition.test(e)) {
                System.out.println("Primary failed, using fallback: " + e.getMessage());
                return fallback.get();
            }
            throw e;
        }
    }
}

// Usage with multiple fallbacks
public class ResilientDataService {
    public Data getData(String key) {
        // Primary: Database
        // Fallback 1: Cache
        // Fallback 2: Default value
        
        try {
            return database.get(key);
        } catch (DatabaseException e) {
            try {
                return cache.get(key);
            } catch (CacheException ce) {
                return getDefaultData();
            }
        }
    }
}
```

---

### 5. Bulkhead Pattern

```java
// Isolate failures to prevent cascade
public class BulkheadExecutor {
    private final Semaphore semaphore;
    private final String name;
    
    public BulkheadExecutor(String name, int maxConcurrent) {
        this.name = name;
        this.semaphore = new Semaphore(maxConcurrent);
    }
    
    public <T> T execute(Supplier<T> operation, long timeoutMs) {
        boolean acquired = false;
        try {
            acquired = semaphore.tryAcquire(timeoutMs, TimeUnit.MILLISECONDS);
            if (!acquired) {
                throw new BulkheadFullException(
                    "Bulkhead '" + name + "' is full. Max concurrent: " + 
                    semaphore.availablePermits());
            }
            return operation.get();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("Interrupted", e);
        } finally {
            if (acquired) {
                semaphore.release();
            }
        }
    }
}

class BulkheadFullException extends RuntimeException {
    public BulkheadFullException(String message) {
        super(message);
    }
}

// Usage - Isolate different services
class OrderService {
    private final BulkheadExecutor paymentBulkhead = new BulkheadExecutor("payment", 10);
    private final BulkheadExecutor inventoryBulkhead = new BulkheadExecutor("inventory", 20);
    
    public void processOrder(Order order) {
        // Payment calls limited to 10 concurrent
        PaymentResult payment = paymentBulkhead.execute(
            () -> paymentService.process(order), 5000);
        
        // Inventory calls limited to 20 concurrent
        inventoryBulkhead.execute(
            () -> inventoryService.reserve(order), 5000);
    }
}
```

---

### 6. Complete Resilience Wrapper

```java
public class ResilientExecutor<T> {
    private final Supplier<T> operation;
    private final Supplier<T> fallback;
    private final CircuitBreaker circuitBreaker;
    private final RetryExecutor retryExecutor;
    private final int timeoutMs;
    
    private ResilientExecutor(Builder<T> builder) {
        this.operation = builder.operation;
        this.fallback = builder.fallback;
        this.circuitBreaker = builder.circuitBreaker;
        this.retryExecutor = builder.retryExecutor;
        this.timeoutMs = builder.timeoutMs;
    }
    
    public T execute() {
        try {
            // Circuit breaker wraps everything
            return circuitBreaker.execute(() -> {
                // Retry with timeout
                return retryExecutor.execute(
                    () -> executeWithTimeout(operation, timeoutMs),
                    e -> e instanceof TimeoutException
                );
            });
        } catch (Exception e) {
            if (fallback != null) {
                return fallback.get();
            }
            throw e;
        }
    }
    
    private T executeWithTimeout(Supplier<T> op, int timeout) {
        // Timeout implementation
        return op.get();  // Simplified
    }
    
    public static <T> Builder<T> builder() {
        return new Builder<>();
    }
    
    public static class Builder<T> {
        private Supplier<T> operation;
        private Supplier<T> fallback;
        private CircuitBreaker circuitBreaker = new CircuitBreaker(5, 30000);
        private RetryExecutor retryExecutor = new RetryExecutor(3, 1000, 2.0);
        private int timeoutMs = 5000;
        
        public Builder<T> operation(Supplier<T> operation) {
            this.operation = operation;
            return this;
        }
        
        public Builder<T> fallback(Supplier<T> fallback) {
            this.fallback = fallback;
            return this;
        }
        
        public Builder<T> circuitBreaker(int threshold, long resetMs) {
            this.circuitBreaker = new CircuitBreaker(threshold, resetMs);
            return this;
        }
        
        public Builder<T> retry(int attempts, long delayMs) {
            this.retryExecutor = new RetryExecutor(attempts, delayMs, 2.0);
            return this;
        }
        
        public Builder<T> timeout(int ms) {
            this.timeoutMs = ms;
            return this;
        }
        
        public ResilientExecutor<T> build() {
            return new ResilientExecutor<>(this);
        }
    }
}

// Usage
ResilientExecutor<Data> executor = ResilientExecutor.<Data>builder()
    .operation(() -> externalService.fetchData())
    .fallback(() -> cache.getData())
    .circuitBreaker(5, 30000)
    .retry(3, 1000)
    .timeout(5000)
    .build();

Data data = executor.execute();
```

---

## Error Handling Summary

| Pattern | Purpose |
|---------|---------|
| **Custom Exceptions** | Clear, meaningful error types |
| **Result Object** | Explicit success/failure handling |
| **Retry** | Handle transient failures |
| **Circuit Breaker** | Prevent cascade failures |
| **Timeout** | Prevent indefinite waits |
| **Fallback** | Graceful degradation |
| **Bulkhead** | Isolate failures |

---

## Key Principles

1. **Fail Fast** - Detect and report errors quickly
2. **Fail Gracefully** - Provide fallbacks and good UX
3. **Log Everything** - Detailed logs for debugging
4. **Don't Swallow Exceptions** - Handle or propagate, never ignore
5. **Use Specific Exceptions** - Avoid generic `catch (Exception e)`
6. **Design for Failure** - Assume external services will fail

---

**Next: Best Practices →**
