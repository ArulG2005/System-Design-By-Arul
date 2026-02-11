# Introduction to Low Level Design (LLD)

## What is Low Level Design?

**Low Level Design (LLD)** is the process of designing the internal structure of a software system. It focuses on:
- Classes and their relationships
- Methods and their implementations
- Data structures and algorithms
- How different components interact with each other

Think of it like this:
- **High Level Design (HLD)** = Blueprint of a house (rooms, floors, layout)
- **Low Level Design (LLD)** = Detailed construction plan (wiring, plumbing, materials)

---

## Why is LLD Important?

### 1. **Code Quality**
Good LLD leads to clean, maintainable, and scalable code.

### 2. **Interview Success**
Top tech companies (Google, Amazon, Microsoft, Meta) ask LLD questions in interviews.

### 3. **Real-World Impact**
Well-designed systems are easier to:
- Debug
- Extend with new features
- Test
- Understand by new team members

---

## LLD vs HLD - Key Differences

| Aspect | High Level Design | Low Level Design |
|--------|------------------|------------------|
| Focus | System architecture | Class/method design |
| Components | Services, databases, APIs | Classes, interfaces, methods |
| Who does it? | System architects | Software developers |
| Diagrams | System diagrams, data flow | Class diagrams, sequence diagrams |
| Example | "Use microservices with Redis cache" | "Create UserService class with login() method" |

---

## What You'll Learn in LLD

### 1. **SOLID Principles**
Five fundamental principles for writing clean code:
- Single Responsibility
- Open/Closed
- Liskov Substitution
- Interface Segregation
- Dependency Inversion

### 2. **Design Patterns**
Reusable solutions to common problems:
- **Creational** - How to create objects
- **Structural** - How to compose objects
- **Behavioral** - How objects communicate

### 3. **UML Diagrams**
Visual representation of your design:
- Class diagrams
- Sequence diagrams
- Use case diagrams

### 4. **Multithreading**
Handling concurrent operations:
- Thread management
- Synchronization
- Common problems and solutions

---

## How to Approach LLD Problems

### Step 1: Clarify Requirements
Ask questions to understand:
- What are the main features?
- Who are the users?
- What are the constraints?

### Step 2: Identify Core Objects
Find the main entities in the system.

**Example: Parking Lot System**
- ParkingLot
- ParkingSpot
- Vehicle
- Ticket
- Payment

### Step 3: Define Relationships
How do objects interact?
- Vehicle **parks in** ParkingSpot
- ParkingLot **has many** ParkingSpots
- Ticket **belongs to** Vehicle

### Step 4: Apply Design Patterns
Use appropriate patterns to solve specific problems.

### Step 5: Write Clean Code
Follow SOLID principles and best practices.

---

## Example: Simple Library System

Let's see a basic example of LLD thinking:

### Requirements
- Users can borrow books
- Books have authors
- Library tracks which books are available

### Identify Classes

```java
// Core entities
class Book {
    private String isbn;
    private String title;
    private Author author;
    private boolean isAvailable;
}

class Author {
    private String name;
    private List<Book> books;
}

class User {
    private String userId;
    private String name;
    private List<Book> borrowedBooks;
}

class Library {
    private List<Book> books;
    private List<User> users;
    
    public boolean borrowBook(User user, Book book) {
        // Logic here
    }
    
    public void returnBook(User user, Book book) {
        // Logic here
    }
}
```

### Define Relationships
```
User ----borrows----> Book
Book ----written by----> Author
Library ----contains----> Books
Library ----has members----> Users
```

---

## Key Mindset for LLD

### 1. **Think in Objects**
Everything is an object with properties and behaviors.

### 2. **Favor Composition over Inheritance**
Use "has-a" relationships more than "is-a" relationships.

### 3. **Code to Interfaces**
Program to abstractions, not concrete implementations.

### 4. **Keep it Simple**
Don't over-engineer. Start simple and refactor.

### 5. **Think About Change**
Design systems that can easily adapt to new requirements.

---

## Common LLD Interview Questions

1. **Parking Lot System**
2. **Library Management System**
3. **Movie Ticket Booking (BookMyShow)**
4. **Elevator System**
5. **Chess Game**
6. **Tic-Tac-Toe**
7. **Vending Machine**
8. **ATM Machine**
9. **Hotel Booking System**
10. **Amazon/Flipkart Shopping Cart**
11. **Snake and Ladder Game**
12. **File System**
13. **Splitwise (Expense Sharing)**
14. **Cache System (LRU Cache)**
15. **Rate Limiter**

---

## Learning Path

```
Week 1-2: SOLID Principles
    ↓
Week 3-4: Design Patterns (Creational)
    ↓
Week 5-6: Design Patterns (Structural + Behavioral)
    ↓
Week 7-8: Multithreading & Concurrency
    ↓
Week 9-10: Practice LLD Problems
    ↓
Week 11-12: Mock Interviews
```

---

## Tools for LLD

1. **UML Tools**: draw.io, Lucidchart, PlantUML
2. **IDEs**: IntelliJ IDEA, VS Code, Eclipse
3. **Practice**: LeetCode (Premium), GitHub projects

---

## Summary

- LLD is about designing the internal structure of software
- Focus on classes, methods, and their interactions
- Follow SOLID principles for clean code
- Use design patterns for common problems
- Practice regularly with real-world systems

**Next: Software Design Principles →**
