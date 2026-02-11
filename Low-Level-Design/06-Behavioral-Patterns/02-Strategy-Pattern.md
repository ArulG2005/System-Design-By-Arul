# Strategy Design Pattern

## Intent

> **Define a family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from clients that use it.**

---

## The Problem

You have multiple ways to do something:
- Different payment methods (Credit Card, PayPal, UPI)
- Different sorting algorithms (QuickSort, MergeSort, BubbleSort)
- Different compression formats (ZIP, RAR, 7z)

### Bad Approach: Giant if-else

```java
// This is TERRIBLE code! ❌
public void processPayment(String type, double amount) {
    if (type.equals("CREDIT_CARD")) {
        // 50 lines of credit card logic
    } else if (type.equals("PAYPAL")) {
        // 50 lines of PayPal logic
    } else if (type.equals("UPI")) {
        // 50 lines of UPI logic
    } else if (type.equals("BITCOIN")) {
        // Adding new types means modifying this method!
    }
    // Violates Open-Closed Principle!
}
```

---

## Simple Analogy

Think of **Transportation to Work**:
- Strategy 1: Drive a car
- Strategy 2: Take the bus
- Strategy 3: Ride a bike
- Strategy 4: Walk

You can **switch strategies** based on:
- Weather
- Traffic
- How late you are

Each strategy achieves the same goal (get to work) differently!

---

## Structure

```
┌─────────────────────────────────────────┐
│               Context                   │
├─────────────────────────────────────────┤
│ - strategy: Strategy                    │
├─────────────────────────────────────────┤
│ + setStrategy(Strategy)                 │
│ + executeStrategy()                     │
└────────────────────┬────────────────────┘
                     │ uses
                     ▼
┌─────────────────────────────────────────┐
│              «interface»                │
│               Strategy                  │
├─────────────────────────────────────────┤
│ + execute()                             │
└─────────────────────△───────────────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
┌───────────────┐┌───────────────┐┌───────────────┐
│ StrategyA     ││ StrategyB     ││ StrategyC     │
├───────────────┤├───────────────┤├───────────────┤
│ + execute()   ││ + execute()   ││ + execute()   │
└───────────────┘└───────────────┘└───────────────┘
```

---

## Basic Example: Payment Processing

```java
// Strategy interface
interface PaymentStrategy {
    boolean pay(double amount);
    String getPaymentType();
}

// Concrete Strategies
class CreditCardPayment implements PaymentStrategy {
    private String cardNumber;
    private String name;
    
    public CreditCardPayment(String cardNumber, String name) {
        this.cardNumber = cardNumber;
        this.name = name;
    }
    
    @Override
    public boolean pay(double amount) {
        System.out.println("Paying $" + amount + " using Credit Card");
        System.out.println("Card: ****" + cardNumber.substring(12));
        // Credit card processing logic
        return true;
    }
    
    @Override
    public String getPaymentType() {
        return "Credit Card";
    }
}

class PayPalPayment implements PaymentStrategy {
    private String email;
    
    public PayPalPayment(String email) {
        this.email = email;
    }
    
    @Override
    public boolean pay(double amount) {
        System.out.println("Paying $" + amount + " using PayPal");
        System.out.println("Account: " + email);
        // PayPal processing logic
        return true;
    }
    
    @Override
    public String getPaymentType() {
        return "PayPal";
    }
}

class UPIPayment implements PaymentStrategy {
    private String upiId;
    
    public UPIPayment(String upiId) {
        this.upiId = upiId;
    }
    
    @Override
    public boolean pay(double amount) {
        System.out.println("Paying $" + amount + " using UPI");
        System.out.println("UPI ID: " + upiId);
        // UPI processing logic
        return true;
    }
    
    @Override
    public String getPaymentType() {
        return "UPI";
    }
}

class CryptoPayment implements PaymentStrategy {
    private String walletAddress;
    
    public CryptoPayment(String walletAddress) {
        this.walletAddress = walletAddress;
    }
    
    @Override
    public boolean pay(double amount) {
        System.out.println("Paying $" + amount + " using Cryptocurrency");
        System.out.println("Wallet: " + walletAddress.substring(0, 10) + "...");
        return true;
    }
    
    @Override
    public String getPaymentType() {
        return "Crypto";
    }
}

// Context
class ShoppingCart {
    private List<String> items = new ArrayList<>();
    private double total = 0;
    private PaymentStrategy paymentStrategy;  // Strategy reference
    
    public void addItem(String item, double price) {
        items.add(item);
        total += price;
    }
    
    // Set strategy at runtime
    public void setPaymentStrategy(PaymentStrategy strategy) {
        this.paymentStrategy = strategy;
    }
    
    public void checkout() {
        if (paymentStrategy == null) {
            System.out.println("Please select a payment method!");
            return;
        }
        
        System.out.println("\n=== Checkout ===");
        System.out.println("Items: " + items);
        System.out.println("Total: $" + total);
        System.out.println("Payment Method: " + paymentStrategy.getPaymentType());
        System.out.println();
        
        if (paymentStrategy.pay(total)) {
            System.out.println("Payment successful!");
            items.clear();
            total = 0;
        } else {
            System.out.println("Payment failed!");
        }
    }
}

// Usage
public class OnlineStore {
    public static void main(String[] args) {
        ShoppingCart cart = new ShoppingCart();
        cart.addItem("Laptop", 999.99);
        cart.addItem("Mouse", 29.99);
        
        // Customer chooses Credit Card
        cart.setPaymentStrategy(new CreditCardPayment("1234567890123456", "John Doe"));
        cart.checkout();
        
        // Another order with PayPal
        cart.addItem("Keyboard", 79.99);
        cart.setPaymentStrategy(new PayPalPayment("john@email.com"));
        cart.checkout();
        
        // Another order with UPI
        cart.addItem("Headphones", 149.99);
        cart.setPaymentStrategy(new UPIPayment("john@okbank"));
        cart.checkout();
    }
}
```

---

## Real-World Examples

### Example 1: Sorting Algorithms

```java
// Strategy interface
interface SortStrategy {
    void sort(int[] array);
    String getName();
}

// Concrete Strategies
class BubbleSort implements SortStrategy {
    @Override
    public void sort(int[] array) {
        System.out.println("Using Bubble Sort (O(n²))");
        int n = array.length;
        for (int i = 0; i < n - 1; i++) {
            for (int j = 0; j < n - i - 1; j++) {
                if (array[j] > array[j + 1]) {
                    int temp = array[j];
                    array[j] = array[j + 1];
                    array[j + 1] = temp;
                }
            }
        }
    }
    
    @Override
    public String getName() { return "Bubble Sort"; }
}

class QuickSort implements SortStrategy {
    @Override
    public void sort(int[] array) {
        System.out.println("Using Quick Sort (O(n log n))");
        quickSort(array, 0, array.length - 1);
    }
    
    private void quickSort(int[] arr, int low, int high) {
        if (low < high) {
            int pi = partition(arr, low, high);
            quickSort(arr, low, pi - 1);
            quickSort(arr, pi + 1, high);
        }
    }
    
    private int partition(int[] arr, int low, int high) {
        int pivot = arr[high];
        int i = low - 1;
        for (int j = low; j < high; j++) {
            if (arr[j] < pivot) {
                i++;
                int temp = arr[i];
                arr[i] = arr[j];
                arr[j] = temp;
            }
        }
        int temp = arr[i + 1];
        arr[i + 1] = arr[high];
        arr[high] = temp;
        return i + 1;
    }
    
    @Override
    public String getName() { return "Quick Sort"; }
}

class MergeSort implements SortStrategy {
    @Override
    public void sort(int[] array) {
        System.out.println("Using Merge Sort (O(n log n), stable)");
        mergeSort(array, 0, array.length - 1);
    }
    
    private void mergeSort(int[] arr, int l, int r) {
        if (l < r) {
            int m = l + (r - l) / 2;
            mergeSort(arr, l, m);
            mergeSort(arr, m + 1, r);
            merge(arr, l, m, r);
        }
    }
    
    private void merge(int[] arr, int l, int m, int r) {
        // Merge implementation
        int n1 = m - l + 1;
        int n2 = r - m;
        int[] L = new int[n1];
        int[] R = new int[n2];
        
        for (int i = 0; i < n1; i++) L[i] = arr[l + i];
        for (int j = 0; j < n2; j++) R[j] = arr[m + 1 + j];
        
        int i = 0, j = 0, k = l;
        while (i < n1 && j < n2) {
            if (L[i] <= R[j]) arr[k++] = L[i++];
            else arr[k++] = R[j++];
        }
        while (i < n1) arr[k++] = L[i++];
        while (j < n2) arr[k++] = R[j++];
    }
    
    @Override
    public String getName() { return "Merge Sort"; }
}

// Context with strategy selection logic
class Sorter {
    private SortStrategy strategy;
    
    public void setStrategy(SortStrategy strategy) {
        this.strategy = strategy;
    }
    
    // Automatically select best strategy
    public void autoSelectStrategy(int[] array) {
        if (array.length < 10) {
            strategy = new BubbleSort();  // Simple for small arrays
        } else if (needsStability()) {
            strategy = new MergeSort();   // Stable sort
        } else {
            strategy = new QuickSort();   // Fast for large arrays
        }
    }
    
    private boolean needsStability() {
        // Check if stability is required
        return false;
    }
    
    public void sort(int[] array) {
        if (strategy == null) {
            autoSelectStrategy(array);
        }
        strategy.sort(array);
    }
}

// Usage
public class SortingDemo {
    public static void main(String[] args) {
        Sorter sorter = new Sorter();
        
        int[] data1 = {64, 34, 25, 12, 22, 11, 90};
        sorter.setStrategy(new BubbleSort());
        sorter.sort(data1);
        System.out.println("Result: " + Arrays.toString(data1));
        
        int[] data2 = {38, 27, 43, 3, 9, 82, 10};
        sorter.setStrategy(new QuickSort());
        sorter.sort(data2);
        System.out.println("Result: " + Arrays.toString(data2));
    }
}
```

---

### Example 2: Navigation App (Route Strategies)

```java
// Strategy interface
interface RouteStrategy {
    void buildRoute(String from, String to);
    double getEstimatedTime();
    double getDistance();
}

// Concrete Strategies
class CarRoute implements RouteStrategy {
    private double distance;
    private double time;
    
    @Override
    public void buildRoute(String from, String to) {
        System.out.println("🚗 Building CAR route from " + from + " to " + to);
        // Use road network, consider traffic
        distance = 25.5;  // km
        time = 35;        // minutes
        System.out.println("Via: Highway 101, Main Street");
        System.out.println("Toll roads: Yes ($5.00)");
    }
    
    @Override
    public double getEstimatedTime() { return time; }
    
    @Override
    public double getDistance() { return distance; }
}

class WalkingRoute implements RouteStrategy {
    private double distance;
    private double time;
    
    @Override
    public void buildRoute(String from, String to) {
        System.out.println("🚶 Building WALKING route from " + from + " to " + to);
        // Use pedestrian paths, shortcuts
        distance = 4.2;   // km
        time = 52;        // minutes
        System.out.println("Via: City Park, Pedestrian Bridge");
    }
    
    @Override
    public double getEstimatedTime() { return time; }
    
    @Override
    public double getDistance() { return distance; }
}

class PublicTransportRoute implements RouteStrategy {
    private double distance;
    private double time;
    
    @Override
    public void buildRoute(String from, String to) {
        System.out.println("🚌 Building PUBLIC TRANSPORT route from " + from + " to " + to);
        distance = 18.0;  // km
        time = 45;        // minutes
        System.out.println("Via: Bus 42 → Metro Blue Line → Walk 5 min");
        System.out.println("Fare: $2.50");
    }
    
    @Override
    public double getEstimatedTime() { return time; }
    
    @Override
    public double getDistance() { return distance; }
}

class BikeRoute implements RouteStrategy {
    private double distance;
    private double time;
    
    @Override
    public void buildRoute(String from, String to) {
        System.out.println("🚴 Building BIKE route from " + from + " to " + to);
        distance = 8.5;   // km
        time = 25;        // minutes
        System.out.println("Via: Bike Lane on River Road");
        System.out.println("Elevation gain: 50m");
    }
    
    @Override
    public double getEstimatedTime() { return time; }
    
    @Override
    public double getDistance() { return distance; }
}

// Context
class Navigator {
    private RouteStrategy routeStrategy;
    
    public void setRouteStrategy(RouteStrategy strategy) {
        this.routeStrategy = strategy;
    }
    
    public void navigate(String from, String to) {
        System.out.println("\n=== Navigation ===");
        routeStrategy.buildRoute(from, to);
        System.out.println("Distance: " + routeStrategy.getDistance() + " km");
        System.out.println("ETA: " + routeStrategy.getEstimatedTime() + " min");
    }
}

// Usage
public class NavigationApp {
    public static void main(String[] args) {
        Navigator navigator = new Navigator();
        
        // User selects car
        navigator.setRouteStrategy(new CarRoute());
        navigator.navigate("Home", "Office");
        
        // User switches to bike
        navigator.setRouteStrategy(new BikeRoute());
        navigator.navigate("Office", "Gym");
        
        // User takes public transport
        navigator.setRouteStrategy(new PublicTransportRoute());
        navigator.navigate("Gym", "Home");
    }
}
```

---

### Example 3: Compression Strategies

```java
// Strategy interface
interface CompressionStrategy {
    byte[] compress(byte[] data);
    byte[] decompress(byte[] data);
    String getExtension();
}

// Concrete Strategies
class ZipCompression implements CompressionStrategy {
    @Override
    public byte[] compress(byte[] data) {
        System.out.println("Compressing with ZIP algorithm");
        // ZIP compression logic
        return data;
    }
    
    @Override
    public byte[] decompress(byte[] data) {
        System.out.println("Decompressing ZIP file");
        return data;
    }
    
    @Override
    public String getExtension() { return ".zip"; }
}

class RarCompression implements CompressionStrategy {
    @Override
    public byte[] compress(byte[] data) {
        System.out.println("Compressing with RAR algorithm");
        return data;
    }
    
    @Override
    public byte[] decompress(byte[] data) {
        System.out.println("Decompressing RAR file");
        return data;
    }
    
    @Override
    public String getExtension() { return ".rar"; }
}

class GzipCompression implements CompressionStrategy {
    @Override
    public byte[] compress(byte[] data) {
        System.out.println("Compressing with GZIP algorithm");
        return data;
    }
    
    @Override
    public byte[] decompress(byte[] data) {
        System.out.println("Decompressing GZIP file");
        return data;
    }
    
    @Override
    public String getExtension() { return ".gz"; }
}

// Context
class FileCompressor {
    private CompressionStrategy strategy;
    
    public void setStrategy(CompressionStrategy strategy) {
        this.strategy = strategy;
    }
    
    public void compressFile(String filename, byte[] data) {
        byte[] compressed = strategy.compress(data);
        String newFilename = filename + strategy.getExtension();
        System.out.println("Saved as: " + newFilename);
    }
}
```

---

## Strategy with Lambda (Java 8+)

```java
// Functional interface (single method)
@FunctionalInterface
interface DiscountStrategy {
    double calculateDiscount(double amount);
}

class PricingService {
    private DiscountStrategy discountStrategy;
    
    public void setDiscountStrategy(DiscountStrategy strategy) {
        this.discountStrategy = strategy;
    }
    
    public double calculateFinalPrice(double originalPrice) {
        double discount = discountStrategy.calculateDiscount(originalPrice);
        return originalPrice - discount;
    }
}

// Usage with lambdas!
public class LambdaStrategyDemo {
    public static void main(String[] args) {
        PricingService pricing = new PricingService();
        
        // No discount
        pricing.setDiscountStrategy(amount -> 0);
        System.out.println("No discount: $" + pricing.calculateFinalPrice(100));
        
        // 10% discount
        pricing.setDiscountStrategy(amount -> amount * 0.10);
        System.out.println("10% off: $" + pricing.calculateFinalPrice(100));
        
        // $20 flat discount
        pricing.setDiscountStrategy(amount -> 20);
        System.out.println("$20 off: $" + pricing.calculateFinalPrice(100));
        
        // Buy more save more
        pricing.setDiscountStrategy(amount -> {
            if (amount > 100) return amount * 0.20;
            if (amount > 50) return amount * 0.10;
            return 0;
        });
        System.out.println("Tiered: $" + pricing.calculateFinalPrice(150));
    }
}
```

---

## When to Use Strategy Pattern

### ✅ Use When:
1. **Multiple algorithms** for same task
2. Algorithms should be **interchangeable**
3. Want to avoid **giant if-else/switch**
4. Need to **switch behavior at runtime**

### ❌ Don't Use When:
1. Only one or two fixed algorithms
2. Algorithms rarely change
3. Would add unnecessary complexity

---

## Strategy vs Other Patterns

| Pattern | Difference |
|---------|------------|
| **Strategy** | Change algorithm, same goal |
| **State** | Change behavior based on state |
| **Template** | Same algorithm, different steps |
| **Command** | Encapsulate request as object |

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Intent** | Encapsulate algorithms, make interchangeable |
| **Key Idea** | Composition over inheritance |
| **Benefits** | Open/Closed, no if-else, runtime switching |
| **Use Case** | Payment, sorting, routing, formatting |

### Remember:
- **Interface** defines the strategy contract
- **Concrete strategies** implement algorithms
- **Context** uses strategy through interface
- Strategies are **interchangeable** at runtime

---

**Next: Observer Pattern →**
