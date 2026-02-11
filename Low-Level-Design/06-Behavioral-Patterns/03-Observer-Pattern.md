# Observer Design Pattern

## Intent

> **Define a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically.**

---

## The Problem

You have objects that need to know when something changes:
- News subscribers need updates when new article is published
- Stock traders need alerts when price changes
- UI components need to update when data changes

### Bad Approach: Polling

```java
// Bad: Constantly checking for updates ❌
while (true) {
    Price currentPrice = stock.getPrice();
    if (currentPrice != lastPrice) {
        updateDisplay(currentPrice);
        lastPrice = currentPrice;
    }
    Thread.sleep(1000);  // Check every second - wasteful!
}
```

---

## Simple Analogy

Think of **YouTube Subscriptions**:
- You **subscribe** to a channel
- When YouTuber uploads video, you get **notified**
- You can **unsubscribe** anytime
- YouTuber doesn't manually notify each person

Or think of **Newspaper Subscription**:
- Publisher (Subject) maintains subscriber list
- When new edition is ready, all subscribers get copies
- Subscribers can cancel anytime

---

## Structure

```
┌────────────────────────────────────┐
│          «interface»               │
│            Subject                 │
├────────────────────────────────────┤
│ + attach(Observer)                 │
│ + detach(Observer)                 │
│ + notifyObservers()                │
└──────────────────┬─────────────────┘
                   │
                   │                    ┌─────────────────────────┐
┌──────────────────▼─────────────────┐  │     «interface»         │
│        ConcreteSubject             │  │       Observer          │
├────────────────────────────────────┤  ├─────────────────────────┤
│ - observers: List<Observer>        │──│ + update(data)          │
│ - state                            │  └───────────△─────────────┘
├────────────────────────────────────┤              │
│ + getState()                       │              │
│ + setState(state)                  │    ┌─────────┴─────────┐
└────────────────────────────────────┘    │                   │
                                   ┌──────────────┐  ┌──────────────┐
                                   │ ObserverA    │  │ ObserverB    │
                                   ├──────────────┤  ├──────────────┤
                                   │ + update()   │  │ + update()   │
                                   └──────────────┘  └──────────────┘
```

---

## Basic Example: News Publisher

```java
// Observer interface
interface Subscriber {
    void update(String news);
}

// Subject interface
interface Publisher {
    void subscribe(Subscriber subscriber);
    void unsubscribe(Subscriber subscriber);
    void notifySubscribers();
}

// Concrete Subject
class NewsAgency implements Publisher {
    private List<Subscriber> subscribers = new ArrayList<>();
    private String latestNews;
    
    @Override
    public void subscribe(Subscriber subscriber) {
        subscribers.add(subscriber);
        System.out.println("New subscriber added. Total: " + subscribers.size());
    }
    
    @Override
    public void unsubscribe(Subscriber subscriber) {
        subscribers.remove(subscriber);
        System.out.println("Subscriber removed. Total: " + subscribers.size());
    }
    
    @Override
    public void notifySubscribers() {
        for (Subscriber subscriber : subscribers) {
            subscriber.update(latestNews);
        }
    }
    
    // Business method that triggers notification
    public void publishNews(String news) {
        this.latestNews = news;
        System.out.println("\n📰 BREAKING NEWS: " + news);
        notifySubscribers();
    }
}

// Concrete Observers
class EmailSubscriber implements Subscriber {
    private String email;
    
    public EmailSubscriber(String email) {
        this.email = email;
    }
    
    @Override
    public void update(String news) {
        System.out.println("📧 Email to " + email + ": " + news);
    }
}

class SMSSubscriber implements Subscriber {
    private String phone;
    
    public SMSSubscriber(String phone) {
        this.phone = phone;
    }
    
    @Override
    public void update(String news) {
        System.out.println("📱 SMS to " + phone + ": " + news);
    }
}

class AppNotificationSubscriber implements Subscriber {
    private String userId;
    
    public AppNotificationSubscriber(String userId) {
        this.userId = userId;
    }
    
    @Override
    public void update(String news) {
        System.out.println("🔔 Push to " + userId + ": " + news);
    }
}

// Usage
public class NewsApp {
    public static void main(String[] args) {
        NewsAgency newsAgency = new NewsAgency();
        
        // Create subscribers
        Subscriber email1 = new EmailSubscriber("john@email.com");
        Subscriber email2 = new EmailSubscriber("jane@email.com");
        Subscriber sms = new SMSSubscriber("+1234567890");
        Subscriber app = new AppNotificationSubscriber("user123");
        
        // Subscribe
        newsAgency.subscribe(email1);
        newsAgency.subscribe(email2);
        newsAgency.subscribe(sms);
        newsAgency.subscribe(app);
        
        // Publish news - all subscribers notified!
        newsAgency.publishNews("New COVID variant discovered!");
        
        // One user unsubscribes
        newsAgency.unsubscribe(sms);
        
        // Publish again - only 3 notified
        newsAgency.publishNews("Stock market hits record high!");
    }
}
```

---

## Real-World Examples

### Example 1: Stock Price Monitor

```java
// Observer interface
interface StockObserver {
    void onPriceChange(String stockSymbol, double oldPrice, double newPrice);
}

// Subject
class Stock {
    private String symbol;
    private double price;
    private List<StockObserver> observers = new ArrayList<>();
    
    public Stock(String symbol, double initialPrice) {
        this.symbol = symbol;
        this.price = initialPrice;
    }
    
    public void addObserver(StockObserver observer) {
        observers.add(observer);
    }
    
    public void removeObserver(StockObserver observer) {
        observers.remove(observer);
    }
    
    public void setPrice(double newPrice) {
        double oldPrice = this.price;
        this.price = newPrice;
        
        // Notify all observers of price change
        for (StockObserver observer : observers) {
            observer.onPriceChange(symbol, oldPrice, newPrice);
        }
    }
    
    public String getSymbol() { return symbol; }
    public double getPrice() { return price; }
}

// Concrete Observers
class StockDisplay implements StockObserver {
    private String name;
    
    public StockDisplay(String name) {
        this.name = name;
    }
    
    @Override
    public void onPriceChange(String symbol, double oldPrice, double newPrice) {
        double change = newPrice - oldPrice;
        String direction = change >= 0 ? "📈" : "📉";
        System.out.printf("[%s] %s %s: $%.2f → $%.2f (%+.2f)%n", 
            name, direction, symbol, oldPrice, newPrice, change);
    }
}

class PriceAlert implements StockObserver {
    private String stockSymbol;
    private double targetPrice;
    private boolean alertOnRise;
    
    public PriceAlert(String stockSymbol, double targetPrice, boolean alertOnRise) {
        this.stockSymbol = stockSymbol;
        this.targetPrice = targetPrice;
        this.alertOnRise = alertOnRise;
    }
    
    @Override
    public void onPriceChange(String symbol, double oldPrice, double newPrice) {
        if (!symbol.equals(stockSymbol)) return;
        
        if (alertOnRise && newPrice >= targetPrice && oldPrice < targetPrice) {
            System.out.println("🚨 ALERT: " + symbol + " reached $" + targetPrice + "!");
        }
        if (!alertOnRise && newPrice <= targetPrice && oldPrice > targetPrice) {
            System.out.println("🚨 ALERT: " + symbol + " dropped to $" + targetPrice + "!");
        }
    }
}

class AutoTrader implements StockObserver {
    private String stockSymbol;
    private double buyBelow;
    private double sellAbove;
    
    public AutoTrader(String stockSymbol, double buyBelow, double sellAbove) {
        this.stockSymbol = stockSymbol;
        this.buyBelow = buyBelow;
        this.sellAbove = sellAbove;
    }
    
    @Override
    public void onPriceChange(String symbol, double oldPrice, double newPrice) {
        if (!symbol.equals(stockSymbol)) return;
        
        if (newPrice < buyBelow) {
            System.out.println("🤖 AUTO-BUY: Buying " + symbol + " at $" + newPrice);
        }
        if (newPrice > sellAbove) {
            System.out.println("🤖 AUTO-SELL: Selling " + symbol + " at $" + newPrice);
        }
    }
}

// Usage
public class StockMarket {
    public static void main(String[] args) {
        Stock apple = new Stock("AAPL", 150.00);
        Stock google = new Stock("GOOGL", 2800.00);
        
        // Add displays
        StockDisplay mainDisplay = new StockDisplay("Main Display");
        StockDisplay mobileApp = new StockDisplay("Mobile App");
        
        apple.addObserver(mainDisplay);
        apple.addObserver(mobileApp);
        google.addObserver(mainDisplay);
        
        // Add alerts
        apple.addObserver(new PriceAlert("AAPL", 160.00, true));
        apple.addObserver(new PriceAlert("AAPL", 140.00, false));
        
        // Add auto-trader
        apple.addObserver(new AutoTrader("AAPL", 145.00, 165.00));
        
        // Simulate price changes
        System.out.println("=== Market Opens ===\n");
        
        apple.setPrice(152.50);
        System.out.println();
        
        apple.setPrice(161.00);  // Triggers alert!
        System.out.println();
        
        google.setPrice(2850.00);
        System.out.println();
        
        apple.setPrice(168.00);  // Triggers auto-sell!
    }
}
```

---

### Example 2: Event System (Like DOM Events)

```java
// Event class
class Event {
    private String type;
    private Object data;
    private long timestamp;
    
    public Event(String type, Object data) {
        this.type = type;
        this.data = data;
        this.timestamp = System.currentTimeMillis();
    }
    
    public String getType() { return type; }
    public Object getData() { return data; }
    public long getTimestamp() { return timestamp; }
}

// Observer interface
interface EventListener {
    void onEvent(Event event);
}

// Event Emitter (Subject)
class EventEmitter {
    private Map<String, List<EventListener>> listeners = new HashMap<>();
    
    public void on(String eventType, EventListener listener) {
        listeners.computeIfAbsent(eventType, k -> new ArrayList<>()).add(listener);
    }
    
    public void off(String eventType, EventListener listener) {
        List<EventListener> eventListeners = listeners.get(eventType);
        if (eventListeners != null) {
            eventListeners.remove(listener);
        }
    }
    
    public void emit(String eventType, Object data) {
        Event event = new Event(eventType, data);
        List<EventListener> eventListeners = listeners.get(eventType);
        
        if (eventListeners != null) {
            for (EventListener listener : eventListeners) {
                listener.onEvent(event);
            }
        }
    }
    
    // One-time listener
    public void once(String eventType, EventListener listener) {
        EventListener wrapper = new EventListener() {
            @Override
            public void onEvent(Event event) {
                listener.onEvent(event);
                off(eventType, this);  // Remove after first call
            }
        };
        on(eventType, wrapper);
    }
}

// Example: Order System using events
class OrderSystem extends EventEmitter {
    public void createOrder(String orderId, double amount) {
        System.out.println("\n📦 Creating order: " + orderId);
        // Create order logic...
        emit("order:created", Map.of("orderId", orderId, "amount", amount));
    }
    
    public void payOrder(String orderId) {
        System.out.println("💳 Processing payment for: " + orderId);
        // Payment logic...
        emit("order:paid", Map.of("orderId", orderId));
    }
    
    public void shipOrder(String orderId) {
        System.out.println("🚚 Shipping order: " + orderId);
        // Shipping logic...
        emit("order:shipped", Map.of("orderId", orderId));
    }
}

// Usage
public class EventDemo {
    public static void main(String[] args) {
        OrderSystem orderSystem = new OrderSystem();
        
        // Email service listens to events
        orderSystem.on("order:created", event -> {
            Map<String, Object> data = (Map<String, Object>) event.getData();
            System.out.println("📧 Email: Your order " + data.get("orderId") + " is confirmed!");
        });
        
        orderSystem.on("order:paid", event -> {
            Map<String, Object> data = (Map<String, Object>) event.getData();
            System.out.println("📧 Email: Payment received for " + data.get("orderId"));
        });
        
        orderSystem.on("order:shipped", event -> {
            Map<String, Object> data = (Map<String, Object>) event.getData();
            System.out.println("📧 Email: Your order " + data.get("orderId") + " has shipped!");
        });
        
        // Analytics service
        orderSystem.on("order:created", event -> {
            System.out.println("📊 Analytics: New order recorded");
        });
        
        // Inventory service
        orderSystem.on("order:paid", event -> {
            System.out.println("📦 Inventory: Reserving items for order");
        });
        
        // Process an order
        orderSystem.createOrder("ORD-001", 99.99);
        orderSystem.payOrder("ORD-001");
        orderSystem.shipOrder("ORD-001");
    }
}
```

---

### Example 3: Weather Station

```java
// Weather data
class WeatherData {
    private double temperature;
    private double humidity;
    private double pressure;
    
    public WeatherData(double temperature, double humidity, double pressure) {
        this.temperature = temperature;
        this.humidity = humidity;
        this.pressure = pressure;
    }
    
    public double getTemperature() { return temperature; }
    public double getHumidity() { return humidity; }
    public double getPressure() { return pressure; }
}

// Observer interface
interface WeatherObserver {
    void update(WeatherData data);
}

// Subject
class WeatherStation {
    private List<WeatherObserver> observers = new ArrayList<>();
    private WeatherData currentWeather;
    
    public void addObserver(WeatherObserver observer) {
        observers.add(observer);
    }
    
    public void removeObserver(WeatherObserver observer) {
        observers.remove(observer);
    }
    
    public void setMeasurements(double temp, double humidity, double pressure) {
        currentWeather = new WeatherData(temp, humidity, pressure);
        notifyObservers();
    }
    
    private void notifyObservers() {
        for (WeatherObserver observer : observers) {
            observer.update(currentWeather);
        }
    }
}

// Concrete Observers
class CurrentConditionsDisplay implements WeatherObserver {
    @Override
    public void update(WeatherData data) {
        System.out.println("\n=== Current Conditions ===");
        System.out.printf("Temperature: %.1f°C%n", data.getTemperature());
        System.out.printf("Humidity: %.1f%%%n", data.getHumidity());
    }
}

class StatisticsDisplay implements WeatherObserver {
    private List<Double> temperatures = new ArrayList<>();
    
    @Override
    public void update(WeatherData data) {
        temperatures.add(data.getTemperature());
        
        double avg = temperatures.stream()
            .mapToDouble(Double::doubleValue)
            .average()
            .orElse(0);
        double max = temperatures.stream()
            .mapToDouble(Double::doubleValue)
            .max()
            .orElse(0);
        double min = temperatures.stream()
            .mapToDouble(Double::doubleValue)
            .min()
            .orElse(0);
        
        System.out.println("\n=== Temperature Stats ===");
        System.out.printf("Avg/Max/Min: %.1f/%.1f/%.1f°C%n", avg, max, min);
    }
}

class ForecastDisplay implements WeatherObserver {
    private double lastPressure = 0;
    
    @Override
    public void update(WeatherData data) {
        System.out.println("\n=== Forecast ===");
        if (data.getPressure() > lastPressure) {
            System.out.println("Improving weather on the way!");
        } else if (data.getPressure() < lastPressure) {
            System.out.println("Watch out for cooler, rainy weather");
        } else {
            System.out.println("More of the same");
        }
        lastPressure = data.getPressure();
    }
}

// Usage
public class WeatherApp {
    public static void main(String[] args) {
        WeatherStation station = new WeatherStation();
        
        station.addObserver(new CurrentConditionsDisplay());
        station.addObserver(new StatisticsDisplay());
        station.addObserver(new ForecastDisplay());
        
        // Simulate weather changes
        System.out.println("=== Weather Update 1 ===");
        station.setMeasurements(25.5, 65, 1013.1);
        
        System.out.println("\n=== Weather Update 2 ===");
        station.setMeasurements(27.2, 70, 1012.5);
        
        System.out.println("\n=== Weather Update 3 ===");
        station.setMeasurements(23.8, 90, 1009.2);
    }
}
```

---

## Push vs Pull Model

### Push Model (shown above)
Subject sends data to observers:
```java
void update(WeatherData data);  // Data pushed to observer
```

### Pull Model
Observers request data from subject:
```java
void update(WeatherStation station);  // Observer pulls what it needs
// In observer:
public void update(WeatherStation station) {
    double temp = station.getTemperature();  // Pull only what's needed
}
```

---

## Java Built-in Observer (Deprecated)

```java
// Java had Observer/Observable (deprecated in Java 9)
// Use your own implementation or:
// - java.beans.PropertyChangeListener
// - java.util.concurrent.Flow (Reactive Streams)

import java.beans.PropertyChangeListener;
import java.beans.PropertyChangeSupport;

class Stock {
    private PropertyChangeSupport support = new PropertyChangeSupport(this);
    private double price;
    
    public void addPropertyChangeListener(PropertyChangeListener listener) {
        support.addPropertyChangeListener(listener);
    }
    
    public void setPrice(double newPrice) {
        double oldPrice = this.price;
        this.price = newPrice;
        support.firePropertyChange("price", oldPrice, newPrice);
    }
}
```

---

## When to Use Observer Pattern

### ✅ Use When:
1. **One-to-many** relationship between objects
2. Changes in one object should **notify others**
3. Don't know **how many** observers ahead of time
4. **Loose coupling** is needed

### ❌ Don't Use When:
1. Only one or few static subscribers
2. Order of notification matters critically
3. Observers need synchronous response

---

## Observer vs Other Patterns

| Pattern | Comparison |
|---------|------------|
| **Observer** | One-to-many notification |
| **Mediator** | Many-to-many through central hub |
| **Pub-Sub** | More decoupled, often with message broker |
| **Event Bus** | Global event system |

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Intent** | Notify multiple objects of state changes |
| **Key Idea** | Subject maintains list of observers |
| **Benefits** | Loose coupling, dynamic subscribers |
| **Use Case** | News feeds, stock prices, UI updates |

### Remember:
- **Subject** = the thing being observed
- **Observers** = things that react to changes
- Observers can be added/removed at runtime
- Subject knows nothing about concrete observers

---

**Next: Command Pattern →**
