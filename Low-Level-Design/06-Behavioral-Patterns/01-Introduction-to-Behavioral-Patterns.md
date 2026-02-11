# Behavioral Design Patterns - Introduction

## What are Behavioral Patterns?

> **Behavioral patterns are concerned with algorithms and the assignment of responsibilities between objects.**

While **Creational** patterns deal with object creation and **Structural** patterns deal with object composition, **Behavioral** patterns focus on:
- **Communication** between objects
- **Algorithms** and responsibilities
- **Control flow** and interactions

---

## The 10 Behavioral Patterns

```
┌─────────────────────────────────────────────────────────────────┐
│                    BEHAVIORAL PATTERNS                          │
├─────────────────────┬───────────────────────────────────────────┤
│ Strategy            │ Switch algorithms at runtime               │
├─────────────────────┼───────────────────────────────────────────┤
│ Observer            │ Notify multiple objects of state changes   │
├─────────────────────┼───────────────────────────────────────────┤
│ Command             │ Encapsulate requests as objects            │
├─────────────────────┼───────────────────────────────────────────┤
│ Iterator            │ Sequential access without exposing internals│
├─────────────────────┼───────────────────────────────────────────┤
│ Template Method     │ Define algorithm skeleton, defer steps     │
├─────────────────────┼───────────────────────────────────────────┤
│ State               │ Alter behavior when state changes          │
├─────────────────────┼───────────────────────────────────────────┤
│ Chain of            │ Pass request along chain of handlers       │
│ Responsibility      │                                            │
├─────────────────────┼───────────────────────────────────────────┤
│ Mediator            │ Reduce direct communication between objects│
├─────────────────────┼───────────────────────────────────────────┤
│ Memento             │ Capture and restore object state           │
├─────────────────────┼───────────────────────────────────────────┤
│ Visitor             │ Add operations to objects without changing │
│                     │ their classes                              │
└─────────────────────┴───────────────────────────────────────────┘
```

---

## Quick Comparison

| Pattern | When to Use |
|---------|-------------|
| **Strategy** | Multiple algorithms, decide at runtime |
| **Observer** | One-to-many notification system |
| **Command** | Queue, log, or undo operations |
| **Iterator** | Traverse collection without exposing structure |
| **Template Method** | Same algorithm, different steps |
| **State** | Object behavior depends on state |
| **Chain of Responsibility** | Multiple handlers for a request |
| **Mediator** | Many-to-many becomes many-to-one |
| **Memento** | Undo/restore functionality |
| **Visitor** | Add operations to class hierarchy |

---

## Common Interview Patterns

In LLD interviews, these are **most commonly asked**:

1. **Strategy** - Payment processing, sorting algorithms
2. **Observer** - Notification systems, event handling
3. **State** - Vending machines, order status
4. **Command** - Undo/redo, remote controls

---

**Let's explore each pattern in detail! →**
