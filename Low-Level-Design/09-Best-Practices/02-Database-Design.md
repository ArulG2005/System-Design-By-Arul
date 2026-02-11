# Database Design for LLD

## Why Database Design Matters in LLD

Low-Level Design isn't just about code patterns—it includes:
- Data modeling
- Schema design
- Query optimization
- Data integrity

---

## 1. Entity Modeling

### Identify Entities from Requirements

**Example: E-Commerce System**

```
Entities:
- User
- Product
- Category
- Order
- OrderItem
- Payment
- Address
- Review
- Cart
- CartItem
```

### Entity Relationships

```java
// One-to-One: User ↔ Profile
class User {
    private Long id;
    private UserProfile profile;  // One profile per user
}

// One-to-Many: User → Orders
class User {
    private Long id;
    private List<Order> orders;  // One user, many orders
}

// Many-to-Many: Product ↔ Category
class Product {
    private Long id;
    private Set<Category> categories;  // Products belong to multiple categories
}

class Category {
    private Long id;
    private Set<Product> products;  // Categories have multiple products
}
```

---

## 2. Schema Design

### Normalization Levels

```sql
-- First Normal Form (1NF): No repeating groups
-- BAD: Repeating groups ❌
CREATE TABLE orders (
    id INT,
    customer VARCHAR,
    item1 VARCHAR,
    item2 VARCHAR,
    item3 VARCHAR  -- What if 4 items?
);

-- GOOD: Separate table ✅
CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_id INT
);

CREATE TABLE order_items (
    id INT PRIMARY KEY,
    order_id INT REFERENCES orders(id),
    product_id INT,
    quantity INT
);
```

```sql
-- Second Normal Form (2NF): No partial dependencies
-- BAD: Partial dependency ❌
CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    product_name VARCHAR,  -- Depends only on product_id
    quantity INT
);

-- GOOD: Separate product details ✅
CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR
);

CREATE TABLE order_items (
    order_id INT,
    product_id INT REFERENCES products(id),
    quantity INT,
    PRIMARY KEY (order_id, product_id)
);
```

```sql
-- Third Normal Form (3NF): No transitive dependencies
-- BAD: Transitive dependency ❌
CREATE TABLE orders (
    id INT,
    customer_id INT,
    customer_name VARCHAR,  -- Depends on customer_id
    customer_email VARCHAR  -- Depends on customer_id
);

-- GOOD: Reference customer table ✅
CREATE TABLE customers (
    id INT PRIMARY KEY,
    name VARCHAR,
    email VARCHAR
);

CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_id INT REFERENCES customers(id)
);
```

---

## 3. Primary Keys

### Surrogate vs Natural Keys

```sql
-- Natural Key: Real-world identifier
CREATE TABLE users (
    email VARCHAR PRIMARY KEY,  -- Email as key
    name VARCHAR
);

-- Surrogate Key: System-generated (Recommended)
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR UNIQUE NOT NULL,
    name VARCHAR
);
```

**Why Surrogate Keys are Preferred:**
- Natural keys can change (email changes, business rules change)
- Consistent across all tables
- Better join performance
- Simpler foreign key references

### UUID vs Auto-Increment

```java
// Auto-Increment: Simple, sequential
// Pros: Compact, ordered, efficient indexing
// Cons: Predictable, distributed systems conflicts

// UUID: Globally unique
// Pros: No coordination needed, unpredictable
// Cons: Larger storage, index fragmentation
public class Order {
    private UUID id = UUID.randomUUID();
}
```

---

## 4. Foreign Keys & Referential Integrity

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Foreign key with cascade rules
    FOREIGN KEY (user_id) REFERENCES users(id)
        ON DELETE CASCADE      -- Delete orders if user deleted
        ON UPDATE CASCADE      -- Update order if user_id changes
);

-- Or for soft-delete approach
CREATE TABLE orders (
    ...
    FOREIGN KEY (user_id) REFERENCES users(id)
        ON DELETE RESTRICT    -- Prevent deleting user with orders
        ON UPDATE CASCADE
);
```

---

## 5. Indexing Strategies

### When to Add Indexes

```sql
-- Index columns used in:
-- 1. WHERE clauses
-- 2. JOIN conditions
-- 3. ORDER BY clauses
-- 4. GROUP BY clauses

-- Frequently queried columns
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_products_category ON products(category_id);

-- Composite index for common query patterns
-- Query: SELECT * FROM orders WHERE user_id = ? AND status = ?
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- Unique constraint (creates index automatically)
ALTER TABLE users ADD CONSTRAINT uk_users_email UNIQUE (email);
```

### Index Column Order Matters

```sql
-- Composite index on (a, b, c)
-- Supports queries filtered by:
-- ✅ a
-- ✅ a, b
-- ✅ a, b, c
-- ❌ b (index not used)
-- ❌ c (index not used)
-- ❌ b, c (index not used)
```

---

## 6. Common Patterns

### Soft Delete

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    email VARCHAR UNIQUE,
    name VARCHAR,
    deleted_at TIMESTAMP NULL,  -- NULL means not deleted
    is_deleted BOOLEAN DEFAULT FALSE
);

-- Query active users
SELECT * FROM users WHERE is_deleted = FALSE;
```

```java
// Java implementation
class User {
    private Long id;
    private String email;
    private LocalDateTime deletedAt;
    private boolean isDeleted;
    
    public void softDelete() {
        this.isDeleted = true;
        this.deletedAt = LocalDateTime.now();
    }
    
    public void restore() {
        this.isDeleted = false;
        this.deletedAt = null;
    }
}
```

### Audit Trail

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    -- Business fields
    total DECIMAL(10, 2),
    status VARCHAR(20),
    
    -- Audit fields
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by BIGINT,
    updated_at TIMESTAMP,
    updated_by BIGINT,
    version INT DEFAULT 0  -- For optimistic locking
);

-- Separate audit log table
CREATE TABLE audit_log (
    id BIGINT PRIMARY KEY,
    entity_type VARCHAR(50),   -- 'Order', 'User', etc.
    entity_id BIGINT,
    action VARCHAR(20),        -- 'CREATE', 'UPDATE', 'DELETE'
    old_value JSON,
    new_value JSON,
    changed_by BIGINT,
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Status/State Machine

```sql
CREATE TABLE order_status_history (
    id BIGINT PRIMARY KEY,
    order_id BIGINT REFERENCES orders(id),
    from_status VARCHAR(20),
    to_status VARCHAR(20),
    changed_by BIGINT,
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    notes TEXT
);

-- Valid status transitions (enforce in application)
-- PENDING → CONFIRMED → SHIPPED → DELIVERED
-- PENDING → CANCELLED
-- CONFIRMED → CANCELLED
```

---

## 7. Repository Pattern

```java
// Repository interface
public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(Long id);
    List<Order> findByUserId(Long userId);
    List<Order> findByStatus(OrderStatus status);
    List<Order> findByUserIdAndStatus(Long userId, OrderStatus status);
    void delete(Order order);
}

// Implementation
public class JdbcOrderRepository implements OrderRepository {
    private final JdbcTemplate jdbc;
    
    @Override
    public Order save(Order order) {
        if (order.getId() == null) {
            return insert(order);
        } else {
            return update(order);
        }
    }
    
    @Override
    public Optional<Order> findById(Long id) {
        String sql = "SELECT * FROM orders WHERE id = ? AND is_deleted = FALSE";
        try {
            Order order = jdbc.queryForObject(sql, orderRowMapper(), id);
            return Optional.of(order);
        } catch (EmptyResultDataAccessException e) {
            return Optional.empty();
        }
    }
    
    @Override
    public List<Order> findByUserIdAndStatus(Long userId, OrderStatus status) {
        String sql = """
            SELECT * FROM orders 
            WHERE user_id = ? AND status = ? AND is_deleted = FALSE
            ORDER BY created_at DESC
            """;
        return jdbc.query(sql, orderRowMapper(), userId, status.name());
    }
    
    private RowMapper<Order> orderRowMapper() {
        return (rs, rowNum) -> {
            Order order = new Order();
            order.setId(rs.getLong("id"));
            order.setUserId(rs.getLong("user_id"));
            order.setStatus(OrderStatus.valueOf(rs.getString("status")));
            order.setTotal(rs.getBigDecimal("total"));
            order.setCreatedAt(rs.getTimestamp("created_at").toLocalDateTime());
            return order;
        };
    }
}
```

---

## 8. Query Optimization

### N+1 Problem

```java
// BAD: N+1 queries ❌
List<Order> orders = orderRepository.findAll();  // 1 query
for (Order order : orders) {
    User user = userRepository.findById(order.getUserId());  // N queries
}

// GOOD: Join or batch fetch ✅
String sql = """
    SELECT o.*, u.name as user_name, u.email as user_email
    FROM orders o
    JOIN users u ON o.user_id = u.id
    """;

// Or fetch users in batch
List<Order> orders = orderRepository.findAll();
Set<Long> userIds = orders.stream()
    .map(Order::getUserId)
    .collect(Collectors.toSet());
Map<Long, User> users = userRepository.findByIds(userIds);  // 1 query for all users
```

### Pagination

```java
// Repository method
public List<Order> findOrders(int page, int size) {
    int offset = page * size;
    String sql = """
        SELECT * FROM orders 
        ORDER BY created_at DESC
        LIMIT ? OFFSET ?
        """;
    return jdbc.query(sql, rowMapper(), size, offset);
}

// Usage
List<Order> firstPage = findOrders(0, 20);   // First 20
List<Order> secondPage = findOrders(1, 20);  // Next 20
```

---

## 9. Data Transfer Objects (DTOs)

```java
// Entity (database representation)
@Entity
public class Order {
    @Id
    private Long id;
    private Long userId;
    private BigDecimal total;
    private OrderStatus status;
    
    @OneToMany
    private List<OrderItem> items;
    
    @ManyToOne
    private User user;
}

// DTO (API representation)
public class OrderDTO {
    private Long id;
    private String userEmail;
    private String userName;
    private BigDecimal total;
    private String status;
    private List<OrderItemDTO> items;
}

// Mapper
public class OrderMapper {
    public OrderDTO toDTO(Order order) {
        OrderDTO dto = new OrderDTO();
        dto.setId(order.getId());
        dto.setUserEmail(order.getUser().getEmail());
        dto.setUserName(order.getUser().getName());
        dto.setTotal(order.getTotal());
        dto.setStatus(order.getStatus().name());
        dto.setItems(order.getItems().stream()
            .map(this::toItemDTO)
            .collect(Collectors.toList()));
        return dto;
    }
}
```

---

## Database Design Checklist

| Check | Question |
|-------|----------|
| ✅ **Normalization** | Is data normalized appropriately? |
| ✅ **Keys** | Are primary and foreign keys defined? |
| ✅ **Indexes** | Are common query patterns indexed? |
| ✅ **Constraints** | Are NOT NULL, UNIQUE enforced? |
| ✅ **Audit** | Are created_at, updated_at tracked? |
| ✅ **Soft Delete** | Is deletion reversible? |
| ✅ **N+1 Prevention** | Are queries optimized? |

---

**Next: Interview Approach →**
