# Flyweight Design Pattern

## Intent

> **Use sharing to support large numbers of fine-grained objects efficiently.**

---

## The Problem

You need **millions of objects** in memory:
- Each object takes up memory
- Many objects share similar data
- System runs out of memory
- Performance degrades

### Example
A forest game with 1,000,000 trees:
- Each tree has: type, color, texture (same for same tree type)
- Each tree has: x, y position (unique)
- If each tree stores everything = **MASSIVE** memory usage

---

## Simple Analogy

Think of **Font Characters** in a document:
- Document has 100,000 "A" characters
- Do we store 100,000 separate "A" objects? **NO!**
- Store ONE "A" with its shape/design (flyweight)
- Store positions separately (context)

Or think of **Chess Pieces**:
- There are only 12 types of pieces (6 × 2 colors)
- But a game database has millions of positions
- Share piece objects, vary only positions

---

## Key Concept: Intrinsic vs Extrinsic State

```
┌────────────────────────────────────────────────────────────────┐
│                      FLYWEIGHT STATES                          │
├───────────────────────────┬────────────────────────────────────┤
│     INTRINSIC STATE       │       EXTRINSIC STATE              │
├───────────────────────────┼────────────────────────────────────┤
│ Shared across objects     │ Unique to each object              │
│ Stored IN flyweight       │ Stored OUTSIDE flyweight           │
│ Context-independent       │ Context-dependent                  │
│ Immutable                 │ Can vary                           │
├───────────────────────────┼────────────────────────────────────┤
│ Example (Tree):           │ Example (Tree):                    │
│ - Type, Color, Texture    │ - X, Y position                    │
│ - Bark pattern            │ - Age, Height                      │
└───────────────────────────┴────────────────────────────────────┘
```

---

## Structure

```
┌──────────────────────────────────────┐
│         FlyweightFactory             │
├──────────────────────────────────────┤
│ - flyweights: Map<Key, Flyweight>    │
├──────────────────────────────────────┤
│ + getFlyweight(key): Flyweight       │
└───────────────────┬──────────────────┘
                    │ creates/reuses
                    ▼
┌──────────────────────────────────────┐
│            Flyweight                 │
├──────────────────────────────────────┤
│ - intrinsicState (shared)            │
├──────────────────────────────────────┤
│ + operation(extrinsicState)          │
└──────────────────────────────────────┘
                    ▲
                    │ uses
┌──────────────────────────────────────┐
│             Client                   │
│ - extrinsicState (unique per object) │
└──────────────────────────────────────┘
```

---

## Basic Example: Character Rendering

```java
// Flyweight - shared character data
class CharacterFlyweight {
    private final char character;      // Intrinsic
    private final String fontFamily;   // Intrinsic
    private final int fontSize;        // Intrinsic
    
    public CharacterFlyweight(char character, String fontFamily, int fontSize) {
        this.character = character;
        this.fontFamily = fontFamily;
        this.fontSize = fontSize;
        System.out.println("Creating new flyweight: '" + character + "' " + 
                          fontFamily + " " + fontSize);
    }
    
    // Extrinsic state passed as parameters
    public void render(int x, int y, String color) {
        System.out.println("Rendering '" + character + "' at (" + x + "," + y + 
                          ") in " + color + " using " + fontFamily + " " + fontSize);
    }
}

// Flyweight Factory - manages shared flyweights
class CharacterFactory {
    private static Map<String, CharacterFlyweight> flyweights = new HashMap<>();
    
    public static CharacterFlyweight getCharacter(char c, String font, int size) {
        String key = c + "_" + font + "_" + size;
        
        if (!flyweights.containsKey(key)) {
            flyweights.put(key, new CharacterFlyweight(c, font, size));
        }
        
        return flyweights.get(key);
    }
    
    public static int getFlyweightCount() {
        return flyweights.size();
    }
}

// Usage
public class TextEditor {
    public static void main(String[] args) {
        // Document: "HELLO" rendered at different positions
        String text = "HELLO";
        
        for (int i = 0; i < text.length(); i++) {
            char c = text.charAt(i);
            CharacterFlyweight charFlyweight = CharacterFactory.getCharacter(
                c, "Arial", 12
            );
            charFlyweight.render(i * 10, 0, "black");
        }
        
        System.out.println("\n--- Another HELLO ---\n");
        
        // Same text again - reuses flyweights!
        for (int i = 0; i < text.length(); i++) {
            char c = text.charAt(i);
            CharacterFlyweight charFlyweight = CharacterFactory.getCharacter(
                c, "Arial", 12
            );
            charFlyweight.render(i * 10, 20, "blue");
        }
        
        // Only 4 flyweights created (H, E, L, O)
        // L is reused for both L's in HELLO
        System.out.println("\nTotal flyweights created: " + 
                          CharacterFactory.getFlyweightCount());
    }
}

/* Output:
Creating new flyweight: 'H' Arial 12
Rendering 'H' at (0,0) in black using Arial 12
Creating new flyweight: 'E' Arial 12
Rendering 'E' at (10,0) in black using Arial 12
Creating new flyweight: 'L' Arial 12
Rendering 'L' at (20,0) in black using Arial 12
Rendering 'L' at (30,0) in black using Arial 12    <-- Reused!
Creating new flyweight: 'O' Arial 12
Rendering 'O' at (40,0) in black using Arial 12

--- Another HELLO ---

Rendering 'H' at (0,20) in blue using Arial 12     <-- All reused!
Rendering 'E' at (10,20) in blue using Arial 12
Rendering 'L' at (20,20) in blue using Arial 12
Rendering 'L' at (30,20) in blue using Arial 12
Rendering 'O' at (40,20) in blue using Arial 12

Total flyweights created: 4
*/
```

---

## Real-World Examples

### Example 1: Forest Game (Trees)

```java
// Flyweight - Tree type (shared)
class TreeType {
    private final String name;        // Intrinsic
    private final String color;       // Intrinsic
    private final String texture;     // Intrinsic (imagine this is large image data)
    
    public TreeType(String name, String color, String texture) {
        this.name = name;
        this.color = color;
        this.texture = texture;
        System.out.println("Loading tree type: " + name + " (takes memory...)");
    }
    
    // Draw with extrinsic state (position)
    public void draw(int x, int y, int age) {
        System.out.println("Drawing " + name + " tree at (" + x + "," + y + 
                          ") age: " + age + " years");
    }
    
    public String getName() { return name; }
}

// Flyweight Factory
class TreeTypeFactory {
    private static Map<String, TreeType> treeTypes = new HashMap<>();
    
    public static TreeType getTreeType(String name, String color, String texture) {
        String key = name + "_" + color;
        
        if (!treeTypes.containsKey(key)) {
            treeTypes.put(key, new TreeType(name, color, texture));
        }
        
        return treeTypes.get(key);
    }
    
    public static int getTypeCount() {
        return treeTypes.size();
    }
}

// Context class - stores extrinsic state
class Tree {
    private int x;          // Extrinsic
    private int y;          // Extrinsic
    private int age;        // Extrinsic
    private TreeType type;  // Reference to flyweight
    
    public Tree(int x, int y, int age, TreeType type) {
        this.x = x;
        this.y = y;
        this.age = age;
        this.type = type;
    }
    
    public void draw() {
        type.draw(x, y, age);
    }
}

// Forest manages many trees
class Forest {
    private List<Tree> trees = new ArrayList<>();
    
    public void plantTree(int x, int y, int age, String name, 
                          String color, String texture) {
        TreeType type = TreeTypeFactory.getTreeType(name, color, texture);
        Tree tree = new Tree(x, y, age, type);
        trees.add(tree);
    }
    
    public void draw() {
        for (Tree tree : trees) {
            tree.draw();
        }
    }
    
    public int getTreeCount() {
        return trees.size();
    }
}

// Usage
public class Game {
    public static void main(String[] args) {
        Forest forest = new Forest();
        
        // Plant 10 trees (only 3 types)
        forest.plantTree(10, 20, 5, "Oak", "Green", "oak_texture.png");
        forest.plantTree(15, 25, 8, "Oak", "Green", "oak_texture.png");
        forest.plantTree(100, 50, 3, "Pine", "DarkGreen", "pine_texture.png");
        forest.plantTree(110, 55, 7, "Pine", "DarkGreen", "pine_texture.png");
        forest.plantTree(200, 80, 2, "Birch", "LightGreen", "birch_texture.png");
        forest.plantTree(20, 30, 10, "Oak", "Green", "oak_texture.png");
        forest.plantTree(120, 60, 4, "Pine", "DarkGreen", "pine_texture.png");
        forest.plantTree(210, 85, 6, "Birch", "LightGreen", "birch_texture.png");
        forest.plantTree(25, 35, 12, "Oak", "Green", "oak_texture.png");
        forest.plantTree(215, 90, 9, "Birch", "LightGreen", "birch_texture.png");
        
        System.out.println("\n=== Drawing Forest ===\n");
        forest.draw();
        
        System.out.println("\n=== Memory Stats ===");
        System.out.println("Trees in forest: " + forest.getTreeCount());
        System.out.println("Tree types loaded: " + TreeTypeFactory.getTypeCount());
        System.out.println("Memory saved: " + 
            (forest.getTreeCount() - TreeTypeFactory.getTypeCount()) + 
            " tree textures not duplicated!");
    }
}

/* Output:
Loading tree type: Oak (takes memory...)
Loading tree type: Pine (takes memory...)
Loading tree type: Birch (takes memory...)

=== Drawing Forest ===

Drawing Oak tree at (10,20) age: 5 years
Drawing Oak tree at (15,25) age: 8 years
Drawing Pine tree at (100,50) age: 3 years
Drawing Pine tree at (110,55) age: 7 years
Drawing Birch tree at (200,80) age: 2 years
Drawing Oak tree at (20,30) age: 10 years
Drawing Pine tree at (120,60) age: 4 years
Drawing Birch tree at (210,85) age: 6 years
Drawing Oak tree at (25,35) age: 12 years
Drawing Birch tree at (215,90) age: 9 years

=== Memory Stats ===
Trees in forest: 10
Tree types loaded: 3
Memory saved: 7 tree textures not duplicated!
*/
```

---

### Example 2: Bullet Hell Game (Particles)

```java
// Flyweight - Bullet type
class BulletType {
    private final String sprite;      // Intrinsic - image data
    private final int damage;         // Intrinsic
    private final String color;       // Intrinsic
    
    public BulletType(String sprite, int damage, String color) {
        this.sprite = sprite;
        this.damage = damage;
        this.color = color;
        System.out.println("Loading bullet sprite: " + sprite);
    }
    
    public void render(int x, int y, double angle, double speed) {
        System.out.println(color + " bullet at (" + x + "," + y + 
                          ") angle: " + angle + " speed: " + speed);
    }
    
    public int getDamage() { return damage; }
}

// Factory
class BulletTypeFactory {
    private static Map<String, BulletType> types = new HashMap<>();
    
    public static BulletType getType(String sprite, int damage, String color) {
        String key = sprite + "_" + color;
        if (!types.containsKey(key)) {
            types.put(key, new BulletType(sprite, damage, color));
        }
        return types.get(key);
    }
}

// Context - Individual bullet
class Bullet {
    private int x, y;                 // Extrinsic
    private double angle;             // Extrinsic
    private double speed;             // Extrinsic
    private BulletType type;          // Flyweight reference
    
    public Bullet(int x, int y, double angle, double speed, BulletType type) {
        this.x = x;
        this.y = y;
        this.angle = angle;
        this.speed = speed;
        this.type = type;
    }
    
    public void update() {
        x += speed * Math.cos(angle);
        y += speed * Math.sin(angle);
    }
    
    public void render() {
        type.render(x, y, angle, speed);
    }
}

// Game manages thousands of bullets
class BulletManager {
    private List<Bullet> bullets = new ArrayList<>();
    
    public void spawnBullet(int x, int y, double angle, String sprite, 
                           int damage, String color, double speed) {
        BulletType type = BulletTypeFactory.getType(sprite, damage, color);
        bullets.add(new Bullet(x, y, angle, speed, type));
    }
    
    public void spawnSpiral(int centerX, int centerY, int count, 
                           String sprite, int damage, String color) {
        for (int i = 0; i < count; i++) {
            double angle = (2 * Math.PI / count) * i;
            spawnBullet(centerX, centerY, angle, sprite, damage, color, 5);
        }
    }
    
    public int getBulletCount() { return bullets.size(); }
}

// Usage
public class BulletHellGame {
    public static void main(String[] args) {
        BulletManager manager = new BulletManager();
        
        // Boss spawns 100 bullets in spiral pattern
        System.out.println("=== Boss Attack ===\n");
        manager.spawnSpiral(400, 300, 50, "circle.png", 10, "Red");
        manager.spawnSpiral(400, 300, 50, "diamond.png", 15, "Blue");
        
        System.out.println("\n=== Stats ===");
        System.out.println("Total bullets: " + manager.getBulletCount());
        System.out.println("With Flyweight: Only 2 bullet sprites loaded");
        System.out.println("Without Flyweight: Would load 100 sprites!");
    }
}
```

---

### Example 3: Chess Pieces

```java
// Flyweight - Chess piece type
class ChessPieceType {
    private final String name;       // Intrinsic
    private final String color;      // Intrinsic
    private final String icon;       // Intrinsic (image data)
    
    public ChessPieceType(String name, String color) {
        this.name = name;
        this.color = color;
        this.icon = loadIcon(name, color);
    }
    
    private String loadIcon(String name, String color) {
        System.out.println("Loading icon: " + color + " " + name);
        return color + "_" + name + ".png";
    }
    
    public void display(String position) {
        System.out.println(color + " " + name + " at " + position);
    }
    
    public String getColor() { return color; }
    public String getName() { return name; }
}

// Factory
class ChessPieceFactory {
    private static Map<String, ChessPieceType> pieces = new HashMap<>();
    
    public static ChessPieceType getPiece(String name, String color) {
        String key = color + "_" + name;
        if (!pieces.containsKey(key)) {
            pieces.put(key, new ChessPieceType(name, color));
        }
        return pieces.get(key);
    }
    
    public static int getPieceTypeCount() {
        return pieces.size();
    }
}

// Context - Piece on specific position
class ChessPiece {
    private String position;         // Extrinsic (e.g., "E4")
    private ChessPieceType type;     // Flyweight
    
    public ChessPiece(String position, ChessPieceType type) {
        this.position = position;
        this.type = type;
    }
    
    public void display() {
        type.display(position);
    }
    
    public void moveTo(String newPosition) {
        this.position = newPosition;
    }
}

// Chess game
class ChessBoard {
    private List<ChessPiece> pieces = new ArrayList<>();
    
    public void placePiece(String position, String name, String color) {
        ChessPieceType type = ChessPieceFactory.getPiece(name, color);
        pieces.add(new ChessPiece(position, type));
    }
    
    public void setupBoard() {
        // White pieces
        placePiece("A1", "Rook", "White");
        placePiece("B1", "Knight", "White");
        placePiece("C1", "Bishop", "White");
        placePiece("D1", "Queen", "White");
        placePiece("E1", "King", "White");
        placePiece("F1", "Bishop", "White");
        placePiece("G1", "Knight", "White");
        placePiece("H1", "Rook", "White");
        for (char c = 'A'; c <= 'H'; c++) {
            placePiece(c + "2", "Pawn", "White");
        }
        
        // Black pieces
        placePiece("A8", "Rook", "Black");
        placePiece("B8", "Knight", "Black");
        placePiece("C8", "Bishop", "Black");
        placePiece("D8", "Queen", "Black");
        placePiece("E8", "King", "Black");
        placePiece("F8", "Bishop", "Black");
        placePiece("G8", "Knight", "Black");
        placePiece("H8", "Rook", "Black");
        for (char c = 'A'; c <= 'H'; c++) {
            placePiece(c + "7", "Pawn", "Black");
        }
    }
    
    public void displayBoard() {
        for (ChessPiece piece : pieces) {
            piece.display();
        }
    }
    
    public int getPieceCount() { return pieces.size(); }
}

// Usage
public class ChessGame {
    public static void main(String[] args) {
        ChessBoard board = new ChessBoard();
        
        System.out.println("=== Setting up board ===\n");
        board.setupBoard();
        
        System.out.println("\n=== Memory Stats ===");
        System.out.println("Total pieces on board: " + board.getPieceCount());
        System.out.println("Unique piece types: " + ChessPieceFactory.getPieceTypeCount());
        System.out.println("Memory efficiency: Only " + 
            ChessPieceFactory.getPieceTypeCount() + 
            " icons loaded for " + board.getPieceCount() + " pieces!");
    }
}

/* Output:
=== Setting up board ===

Loading icon: White Rook
Loading icon: White Knight
Loading icon: White Bishop
Loading icon: White Queen
Loading icon: White King
Loading icon: White Pawn
Loading icon: Black Rook
Loading icon: Black Knight
Loading icon: Black Bishop
Loading icon: Black Queen
Loading icon: Black King
Loading icon: Black Pawn

=== Memory Stats ===
Total pieces on board: 32
Unique piece types: 12
Memory efficiency: Only 12 icons loaded for 32 pieces!
*/
```

---

## Java String Pool - Built-in Flyweight!

```java
public class StringPoolExample {
    public static void main(String[] args) {
        // String literals use Flyweight pattern!
        String s1 = "Hello";
        String s2 = "Hello";
        String s3 = new String("Hello");
        String s4 = s3.intern();  // Get from pool
        
        System.out.println(s1 == s2);  // true - same object from pool!
        System.out.println(s1 == s3);  // false - new object
        System.out.println(s1 == s4);  // true - intern() returns pooled string
        
        // Java reuses String objects = Flyweight!
    }
}
```

---

## When to Use Flyweight Pattern

### ✅ Use When:
1. **Large number** of similar objects needed
2. Objects have **shared intrinsic state**
3. **Memory is a constraint**
4. Object identity is not important

### ❌ Don't Use When:
1. Few objects needed
2. Objects are mostly unique (no sharing possible)
3. You need object identity

---

## Flyweight vs Other Patterns

| Pattern | Comparison |
|---------|------------|
| **Flyweight** | Shares intrinsic state, saves memory |
| **Singleton** | Only ONE instance exists |
| **Prototype** | Clones objects (opposite - creates copies) |
| **Object Pool** | Reuses whole objects (not parts) |

---

## Implementation Tips

1. **Immutable flyweights** - intrinsic state should never change
2. **Factory is required** - controls flyweight creation and reuse
3. **Separate states clearly** - intrinsic (shared) vs extrinsic (unique)
4. **Consider thread safety** - use ConcurrentHashMap in factories

```java
// Thread-safe factory
class ThreadSafeFactory {
    private static ConcurrentHashMap<String, Flyweight> flyweights = 
        new ConcurrentHashMap<>();
    
    public static Flyweight get(String key) {
        return flyweights.computeIfAbsent(key, k -> new Flyweight(k));
    }
}
```

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Intent** | Share objects to save memory |
| **Key Idea** | Separate intrinsic (shared) from extrinsic (unique) state |
| **Benefits** | Massive memory savings for many similar objects |
| **Use Case** | Game particles, text characters, tree forests |

### Remember:
- **Intrinsic** = stored in flyweight, shared
- **Extrinsic** = passed as parameter, unique per context
- Factory controls creation and reuse
- Flyweights should be **immutable**

---

**Congratulations! You've completed all Structural Patterns!**

**Next Section: Behavioral Patterns →**
