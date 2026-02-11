# Builder Design Pattern

## Intent

> **Separate the construction of a complex object from its representation, allowing the same construction process to create different representations.**

---

## The Problem

You need to create objects that:
- Have many parameters (telescope constructor problem)
- Have optional and required parameters
- Require step-by-step construction
- Should be immutable after creation

### The Telescopic Constructor Problem:

```java
// ❌ BAD: Too many constructors!
class Pizza {
    public Pizza(String size) { }
    public Pizza(String size, boolean cheese) { }
    public Pizza(String size, boolean cheese, boolean pepperoni) { }
    public Pizza(String size, boolean cheese, boolean pepperoni, boolean mushrooms) { }
    public Pizza(String size, boolean cheese, boolean pepperoni, boolean mushrooms, 
                 boolean onions, boolean olives, boolean bacon) { }
    // This keeps growing!
}

// Usage: Hard to read!
Pizza pizza = new Pizza("Large", true, true, false, true, false, true);
// What do all these booleans mean? 😕
```

---

## Simple Analogy

Think of **ordering at Subway**:
1. Choose bread (required)
2. Choose size (required)
3. Choose meat (optional)
4. Choose vegetables (optional)
5. Choose sauce (optional)
6. Choose extras (optional)

You build your sandwich **step by step**, and at the end, you get a complete sandwich.

---

## Builder Pattern Structure

```
┌──────────────────────┐         ┌──────────────────────┐
│       Director       │         │       Builder        │
├──────────────────────┤         ├──────────────────────┤
│ - builder: Builder   │────────>│ + buildPartA()       │
├──────────────────────┤         │ + buildPartB()       │
│ + construct(): void  │         │ + getResult(): Product│
└──────────────────────┘         └───────────△──────────┘
                                             │
                                 ┌───────────────────────┐
                                 │   ConcreteBuilder     │
                                 ├───────────────────────┤
                                 │ - product: Product    │
                                 │ + buildPartA(): void  │
                                 │ + buildPartB(): void  │
                                 │ + getResult(): Product│
                                 └───────────────────────┘
```

---

## Basic Implementation

### Example: Building a House

```java
// Product
class House {
    private String foundation;
    private String structure;
    private String roof;
    private String interior;
    private boolean hasGarage;
    private boolean hasPool;
    private boolean hasGarden;
    
    // Private constructor - only builder can create
    private House() {}
    
    // Getters...
    
    @Override
    public String toString() {
        return "House{" +
            "foundation='" + foundation + "', " +
            "structure='" + structure + "', " +
            "roof='" + roof + "', " +
            "interior='" + interior + "', " +
            "hasGarage=" + hasGarage + ", " +
            "hasPool=" + hasPool + ", " +
            "hasGarden=" + hasGarden + '}';
    }
    
    // Static inner Builder class
    public static class Builder {
        private House house;
        
        public Builder() {
            house = new House();
        }
        
        public Builder foundation(String foundation) {
            house.foundation = foundation;
            return this;  // Return builder for chaining
        }
        
        public Builder structure(String structure) {
            house.structure = structure;
            return this;
        }
        
        public Builder roof(String roof) {
            house.roof = roof;
            return this;
        }
        
        public Builder interior(String interior) {
            house.interior = interior;
            return this;
        }
        
        public Builder withGarage() {
            house.hasGarage = true;
            return this;
        }
        
        public Builder withPool() {
            house.hasPool = true;
            return this;
        }
        
        public Builder withGarden() {
            house.hasGarden = true;
            return this;
        }
        
        public House build() {
            // Validation
            if (house.foundation == null || house.structure == null) {
                throw new IllegalStateException("Foundation and structure required!");
            }
            return house;
        }
    }
}

// Usage - Clean and readable!
House house = new House.Builder()
    .foundation("Concrete")
    .structure("Wood and Brick")
    .roof("Tiles")
    .interior("Modern")
    .withGarage()
    .withGarden()
    .build();

System.out.println(house);
```

---

## Real-World Examples

### Example 1: User Registration

```java
class User {
    // Required fields
    private final String username;
    private final String email;
    
    // Optional fields
    private final String firstName;
    private final String lastName;
    private final int age;
    private final String phone;
    private final String address;
    private final boolean newsletter;
    
    private User(Builder builder) {
        this.username = builder.username;
        this.email = builder.email;
        this.firstName = builder.firstName;
        this.lastName = builder.lastName;
        this.age = builder.age;
        this.phone = builder.phone;
        this.address = builder.address;
        this.newsletter = builder.newsletter;
    }
    
    // Getters only - object is immutable
    public String getUsername() { return username; }
    public String getEmail() { return email; }
    public String getFirstName() { return firstName; }
    public String getLastName() { return lastName; }
    
    public static class Builder {
        // Required parameters
        private final String username;
        private final String email;
        
        // Optional parameters with defaults
        private String firstName = "";
        private String lastName = "";
        private int age = 0;
        private String phone = "";
        private String address = "";
        private boolean newsletter = false;
        
        // Constructor with required parameters
        public Builder(String username, String email) {
            this.username = username;
            this.email = email;
        }
        
        public Builder firstName(String firstName) {
            this.firstName = firstName;
            return this;
        }
        
        public Builder lastName(String lastName) {
            this.lastName = lastName;
            return this;
        }
        
        public Builder age(int age) {
            this.age = age;
            return this;
        }
        
        public Builder phone(String phone) {
            this.phone = phone;
            return this;
        }
        
        public Builder address(String address) {
            this.address = address;
            return this;
        }
        
        public Builder newsletter(boolean newsletter) {
            this.newsletter = newsletter;
            return this;
        }
        
        public User build() {
            return new User(this);
        }
    }
}

// Usage
User user = new User.Builder("john_doe", "john@email.com")
    .firstName("John")
    .lastName("Doe")
    .age(30)
    .newsletter(true)
    .build();
```

---

### Example 2: HTTP Request Builder

```java
class HttpRequest {
    private final String url;
    private final String method;
    private final Map<String, String> headers;
    private final Map<String, String> queryParams;
    private final String body;
    private final int timeout;
    
    private HttpRequest(Builder builder) {
        this.url = builder.url;
        this.method = builder.method;
        this.headers = new HashMap<>(builder.headers);
        this.queryParams = new HashMap<>(builder.queryParams);
        this.body = builder.body;
        this.timeout = builder.timeout;
    }
    
    public void execute() {
        System.out.println(method + " " + url);
        System.out.println("Headers: " + headers);
        System.out.println("Query: " + queryParams);
        if (body != null) {
            System.out.println("Body: " + body);
        }
    }
    
    public static class Builder {
        private final String url;
        private String method = "GET";
        private Map<String, String> headers = new HashMap<>();
        private Map<String, String> queryParams = new HashMap<>();
        private String body = null;
        private int timeout = 30000;
        
        public Builder(String url) {
            this.url = url;
        }
        
        public Builder get() {
            this.method = "GET";
            return this;
        }
        
        public Builder post() {
            this.method = "POST";
            return this;
        }
        
        public Builder put() {
            this.method = "PUT";
            return this;
        }
        
        public Builder delete() {
            this.method = "DELETE";
            return this;
        }
        
        public Builder header(String key, String value) {
            this.headers.put(key, value);
            return this;
        }
        
        public Builder queryParam(String key, String value) {
            this.queryParams.put(key, value);
            return this;
        }
        
        public Builder body(String body) {
            this.body = body;
            return this;
        }
        
        public Builder timeout(int timeoutMs) {
            this.timeout = timeoutMs;
            return this;
        }
        
        public HttpRequest build() {
            return new HttpRequest(this);
        }
    }
}

// Usage
HttpRequest request = new HttpRequest.Builder("https://api.example.com/users")
    .post()
    .header("Content-Type", "application/json")
    .header("Authorization", "Bearer token123")
    .body("{\"name\": \"John\", \"email\": \"john@email.com\"}")
    .timeout(5000)
    .build();

request.execute();
```

---

### Example 3: SQL Query Builder

```java
class SQLQuery {
    private final String query;
    
    private SQLQuery(String query) {
        this.query = query;
    }
    
    public String getQuery() {
        return query;
    }
    
    @Override
    public String toString() {
        return query;
    }
    
    public static class Builder {
        private StringBuilder query = new StringBuilder();
        private String table;
        private List<String> columns = new ArrayList<>();
        private List<String> conditions = new ArrayList<>();
        private String orderBy;
        private Integer limit;
        
        public Builder select(String... columns) {
            this.columns.addAll(Arrays.asList(columns));
            return this;
        }
        
        public Builder from(String table) {
            this.table = table;
            return this;
        }
        
        public Builder where(String condition) {
            this.conditions.add(condition);
            return this;
        }
        
        public Builder and(String condition) {
            this.conditions.add("AND " + condition);
            return this;
        }
        
        public Builder or(String condition) {
            this.conditions.add("OR " + condition);
            return this;
        }
        
        public Builder orderBy(String column) {
            this.orderBy = column;
            return this;
        }
        
        public Builder limit(int limit) {
            this.limit = limit;
            return this;
        }
        
        public SQLQuery build() {
            query.append("SELECT ");
            
            if (columns.isEmpty()) {
                query.append("*");
            } else {
                query.append(String.join(", ", columns));
            }
            
            query.append(" FROM ").append(table);
            
            if (!conditions.isEmpty()) {
                query.append(" WHERE ");
                query.append(String.join(" ", conditions));
            }
            
            if (orderBy != null) {
                query.append(" ORDER BY ").append(orderBy);
            }
            
            if (limit != null) {
                query.append(" LIMIT ").append(limit);
            }
            
            return new SQLQuery(query.toString());
        }
    }
}

// Usage
SQLQuery query = new SQLQuery.Builder()
    .select("id", "name", "email")
    .from("users")
    .where("age > 18")
    .and("status = 'active'")
    .orderBy("created_at DESC")
    .limit(10)
    .build();

System.out.println(query);
// SELECT id, name, email FROM users WHERE age > 18 AND status = 'active' ORDER BY created_at DESC LIMIT 10
```

---

### Example 4: Pizza Builder (Classic)

```java
class Pizza {
    private final String size;
    private final String crust;
    private final boolean cheese;
    private final boolean pepperoni;
    private final boolean mushrooms;
    private final boolean onions;
    private final boolean bacon;
    private final boolean olives;
    private final String extraNotes;
    
    private Pizza(Builder builder) {
        this.size = builder.size;
        this.crust = builder.crust;
        this.cheese = builder.cheese;
        this.pepperoni = builder.pepperoni;
        this.mushrooms = builder.mushrooms;
        this.onions = builder.onions;
        this.bacon = builder.bacon;
        this.olives = builder.olives;
        this.extraNotes = builder.extraNotes;
    }
    
    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder();
        sb.append(size + " " + crust + " pizza with: ");
        if (cheese) sb.append("cheese, ");
        if (pepperoni) sb.append("pepperoni, ");
        if (mushrooms) sb.append("mushrooms, ");
        if (onions) sb.append("onions, ");
        if (bacon) sb.append("bacon, ");
        if (olives) sb.append("olives, ");
        if (extraNotes != null) sb.append("Notes: " + extraNotes);
        return sb.toString();
    }
    
    public static class Builder {
        // Required
        private final String size;
        
        // Optional with defaults
        private String crust = "regular";
        private boolean cheese = true;
        private boolean pepperoni = false;
        private boolean mushrooms = false;
        private boolean onions = false;
        private boolean bacon = false;
        private boolean olives = false;
        private String extraNotes = null;
        
        public Builder(String size) {
            this.size = size;
        }
        
        public Builder crust(String crust) {
            this.crust = crust;
            return this;
        }
        
        public Builder noCheese() {
            this.cheese = false;
            return this;
        }
        
        public Builder pepperoni() {
            this.pepperoni = true;
            return this;
        }
        
        public Builder mushrooms() {
            this.mushrooms = true;
            return this;
        }
        
        public Builder onions() {
            this.onions = true;
            return this;
        }
        
        public Builder bacon() {
            this.bacon = true;
            return this;
        }
        
        public Builder olives() {
            this.olives = true;
            return this;
        }
        
        public Builder notes(String notes) {
            this.extraNotes = notes;
            return this;
        }
        
        public Pizza build() {
            return new Pizza(this);
        }
    }
}

// Usage - Very readable!
Pizza pizza = new Pizza.Builder("Large")
    .crust("thin")
    .pepperoni()
    .mushrooms()
    .bacon()
    .notes("Extra crispy please")
    .build();

System.out.println(pizza);
// Large thin pizza with: cheese, pepperoni, mushrooms, bacon, Notes: Extra crispy please
```

---

## Director Pattern (Optional)

The Director encapsulates a particular construction process:

```java
class HouseDirector {
    private House.Builder builder;
    
    public HouseDirector(House.Builder builder) {
        this.builder = builder;
    }
    
    // Predefined configurations
    public House constructSimpleHouse() {
        return builder
            .foundation("Concrete")
            .structure("Wood")
            .roof("Shingles")
            .interior("Basic")
            .build();
    }
    
    public House constructLuxuryHouse() {
        return builder
            .foundation("Reinforced Concrete")
            .structure("Steel and Glass")
            .roof("Solar Tiles")
            .interior("Premium")
            .withGarage()
            .withPool()
            .withGarden()
            .build();
    }
    
    public House constructCabin() {
        return builder
            .foundation("Stone")
            .structure("Log")
            .roof("Wood")
            .interior("Rustic")
            .build();
    }
}

// Usage
HouseDirector director = new HouseDirector(new House.Builder());
House simple = director.constructSimpleHouse();
House luxury = director.constructLuxuryHouse();
```

---

## Builder in Java Standard Library

### StringBuilder
```java
String result = new StringBuilder()
    .append("Hello")
    .append(" ")
    .append("World")
    .append("!")
    .toString();
```

### Stream.Builder
```java
Stream<String> stream = Stream.<String>builder()
    .add("a")
    .add("b")
    .add("c")
    .build();
```

### Locale.Builder
```java
Locale locale = new Locale.Builder()
    .setLanguage("en")
    .setRegion("US")
    .build();
```

---

## Builder with Lombok

Using Lombok library makes it even simpler:

```java
import lombok.Builder;
import lombok.ToString;

@Builder
@ToString
class User {
    private String username;
    private String email;
    private String firstName;
    private String lastName;
    private int age;
}

// Usage - Lombok generates the builder!
User user = User.builder()
    .username("john_doe")
    .email("john@email.com")
    .firstName("John")
    .age(30)
    .build();
```

---

## When to Use Builder Pattern

### ✅ Use When:
1. **Many constructor parameters** (especially > 4)
2. **Mix of required and optional parameters**
3. **Object should be immutable after creation**
4. **Step-by-step construction needed**
5. **Same construction process, different representations**

### ❌ Don't Use When:
1. Simple object with few parameters
2. Mutable objects (setters are fine)
3. All parameters are required

---

## Builder vs Constructor vs Setter

| Approach | Pros | Cons |
|----------|------|------|
| **Telescopic Constructor** | Simple for few params | Unreadable with many params |
| **JavaBeans (Setters)** | Easy to understand | Mutable, inconsistent state |
| **Builder** | Readable, immutable, flexible | More code to write |

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Intent** | Build complex objects step by step |
| **Key Idea** | Separate construction from representation |
| **Benefits** | Readable code, immutability, validation |
| **Structure** | Product + Builder + (optional) Director |
| **Fluent API** | Method chaining with `return this` |

### Remember:
- Use static inner Builder class
- Return `this` for method chaining
- Make Product constructor private
- Validate in `build()` method
- Consider Director for predefined configurations

---

**Next: Abstract Factory Pattern →**
