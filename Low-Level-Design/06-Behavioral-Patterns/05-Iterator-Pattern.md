# Iterator Design Pattern

## Intent

> **Provide a way to access the elements of an aggregate object sequentially without exposing its underlying representation.**

---

## The Problem

You have a collection and want to:
- Traverse all elements **without knowing** internal structure
- Support **multiple traversal** algorithms
- Have a **uniform** way to iterate different collections
- Iterate **without exposing** implementation details

---

## Simple Analogy

Think of a **Playlist**:
- You can go to next song, previous song
- You don't care if songs are in array, linked list, or database
- Play button just says "give me next song"
- Shuffle mode = different iterator, same playlist

Or think of a **TV Remote Channel Surfing**:
- Press "Next" to go to next channel
- Don't know how channels are stored internally
- Just need "hasNext" and "next"

---

## Structure

```
┌────────────────────────────────────┐     ┌────────────────────────────────────┐
│         «interface»                │     │           «interface»              │
│           Iterable                 │     │            Iterator                │
├────────────────────────────────────┤     ├────────────────────────────────────┤
│ + createIterator(): Iterator       │     │ + hasNext(): boolean               │
└────────────────△───────────────────┘     │ + next(): Element                  │
                 │                         │ + remove(): void                   │
                 │                         └──────────────────△─────────────────┘
┌────────────────┴───────────────────┐                       │
│        ConcreteCollection          │     ┌─────────────────┴──────────────────┐
├────────────────────────────────────┤     │        ConcreteIterator            │
│ - elements                         │     ├────────────────────────────────────┤
├────────────────────────────────────┤     │ - collection                       │
│ + createIterator(): Iterator       │────>│ - currentPosition                  │
│ + add(Element)                     │     ├────────────────────────────────────┤
│ + get(int): Element                │     │ + hasNext(): boolean               │
└────────────────────────────────────┘     │ + next(): Element                  │
                                           └────────────────────────────────────┘
```

---

## Basic Example: Custom Collection

```java
// Iterator interface
interface Iterator<T> {
    boolean hasNext();
    T next();
    void reset();
}

// Aggregate interface
interface Container<T> {
    Iterator<T> createIterator();
}

// Concrete collection
class BookCollection implements Container<String> {
    private String[] books;
    private int size = 0;
    
    public BookCollection(int capacity) {
        books = new String[capacity];
    }
    
    public void addBook(String book) {
        if (size < books.length) {
            books[size++] = book;
        }
    }
    
    public String getBook(int index) {
        return books[index];
    }
    
    public int getSize() {
        return size;
    }
    
    @Override
    public Iterator<String> createIterator() {
        return new BookIterator(this);
    }
}

// Concrete Iterator
class BookIterator implements Iterator<String> {
    private BookCollection collection;
    private int currentIndex = 0;
    
    public BookIterator(BookCollection collection) {
        this.collection = collection;
    }
    
    @Override
    public boolean hasNext() {
        return currentIndex < collection.getSize();
    }
    
    @Override
    public String next() {
        if (hasNext()) {
            return collection.getBook(currentIndex++);
        }
        return null;
    }
    
    @Override
    public void reset() {
        currentIndex = 0;
    }
}

// Usage
public class Library {
    public static void main(String[] args) {
        BookCollection books = new BookCollection(5);
        books.addBook("Design Patterns");
        books.addBook("Clean Code");
        books.addBook("Refactoring");
        books.addBook("The Pragmatic Programmer");
        
        // Iterate without knowing internal structure!
        Iterator<String> iterator = books.createIterator();
        
        System.out.println("Books in library:");
        while (iterator.hasNext()) {
            System.out.println("- " + iterator.next());
        }
        
        // Reset and iterate again
        iterator.reset();
        System.out.println("\nFirst book: " + iterator.next());
    }
}
```

---

## Real-World Examples

### Example 1: Social Network Friends Iterator

```java
// Profile class
class Profile {
    private String id;
    private String name;
    private String email;
    private List<String> friendIds;
    
    public Profile(String id, String name, String email) {
        this.id = id;
        this.name = name;
        this.email = email;
        this.friendIds = new ArrayList<>();
    }
    
    public void addFriend(String friendId) {
        friendIds.add(friendId);
    }
    
    // Getters
    public String getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public List<String> getFriendIds() { return friendIds; }
}

// Iterator interface
interface ProfileIterator {
    boolean hasNext();
    Profile next();
}

// Social Network interface
interface SocialNetwork {
    ProfileIterator createFriendsIterator(String profileId);
    ProfileIterator createCoworkersIterator(String profileId);
}

// Facebook implementation
class Facebook implements SocialNetwork {
    private Map<String, Profile> profiles = new HashMap<>();
    private Map<String, List<String>> coworkers = new HashMap<>();
    
    public void addProfile(Profile profile) {
        profiles.put(profile.getId(), profile);
    }
    
    public void addCoworkers(String profileId, List<String> coworkerIds) {
        coworkers.put(profileId, coworkerIds);
    }
    
    public Profile getProfile(String id) {
        return profiles.get(id);
    }
    
    @Override
    public ProfileIterator createFriendsIterator(String profileId) {
        return new FacebookFriendsIterator(this, profileId);
    }
    
    @Override
    public ProfileIterator createCoworkersIterator(String profileId) {
        return new FacebookCoworkersIterator(this, profileId);
    }
}

// Friends Iterator
class FacebookFriendsIterator implements ProfileIterator {
    private Facebook facebook;
    private String profileId;
    private int currentIndex = 0;
    private List<Profile> friends;
    
    public FacebookFriendsIterator(Facebook facebook, String profileId) {
        this.facebook = facebook;
        this.profileId = profileId;
        this.friends = new ArrayList<>();
        loadFriends();
    }
    
    private void loadFriends() {
        Profile profile = facebook.getProfile(profileId);
        if (profile != null) {
            for (String friendId : profile.getFriendIds()) {
                Profile friend = facebook.getProfile(friendId);
                if (friend != null) {
                    friends.add(friend);
                }
            }
        }
    }
    
    @Override
    public boolean hasNext() {
        return currentIndex < friends.size();
    }
    
    @Override
    public Profile next() {
        if (hasNext()) {
            return friends.get(currentIndex++);
        }
        return null;
    }
}

// Coworkers Iterator (similar structure)
class FacebookCoworkersIterator implements ProfileIterator {
    private Facebook facebook;
    private String profileId;
    private int currentIndex = 0;
    private List<Profile> coworkersList;
    
    public FacebookCoworkersIterator(Facebook facebook, String profileId) {
        this.facebook = facebook;
        this.profileId = profileId;
        this.coworkersList = new ArrayList<>();
        // Load coworkers logic...
    }
    
    @Override
    public boolean hasNext() {
        return currentIndex < coworkersList.size();
    }
    
    @Override
    public Profile next() {
        if (hasNext()) {
            return coworkersList.get(currentIndex++);
        }
        return null;
    }
}

// Spam filter that uses iterator
class SocialSpammer {
    private SocialNetwork network;
    
    public SocialSpammer(SocialNetwork network) {
        this.network = network;
    }
    
    public void sendSpamToFriends(String profileId, String message) {
        ProfileIterator iterator = network.createFriendsIterator(profileId);
        while (iterator.hasNext()) {
            Profile friend = iterator.next();
            sendEmail(friend.getEmail(), message);
        }
    }
    
    public void sendSpamToCoworkers(String profileId, String message) {
        ProfileIterator iterator = network.createCoworkersIterator(profileId);
        while (iterator.hasNext()) {
            Profile coworker = iterator.next();
            sendEmail(coworker.getEmail(), message);
        }
    }
    
    private void sendEmail(String email, String message) {
        System.out.println("Sending to " + email + ": " + message);
    }
}

// Usage
public class SocialApp {
    public static void main(String[] args) {
        Facebook facebook = new Facebook();
        
        Profile john = new Profile("1", "John", "john@email.com");
        Profile jane = new Profile("2", "Jane", "jane@email.com");
        Profile bob = new Profile("3", "Bob", "bob@email.com");
        
        john.addFriend("2");
        john.addFriend("3");
        
        facebook.addProfile(john);
        facebook.addProfile(jane);
        facebook.addProfile(bob);
        
        SocialSpammer spammer = new SocialSpammer(facebook);
        spammer.sendSpamToFriends("1", "Check out this cool product!");
    }
}
```

---

### Example 2: Tree Traversal Iterators

```java
// Tree Node
class TreeNode {
    int value;
    TreeNode left;
    TreeNode right;
    
    public TreeNode(int value) {
        this.value = value;
    }
}

// Tree Iterator interface
interface TreeIterator {
    boolean hasNext();
    Integer next();
}

// In-order Iterator (Left, Root, Right)
class InOrderIterator implements TreeIterator {
    private Stack<TreeNode> stack = new Stack<>();
    
    public InOrderIterator(TreeNode root) {
        pushLeft(root);
    }
    
    private void pushLeft(TreeNode node) {
        while (node != null) {
            stack.push(node);
            node = node.left;
        }
    }
    
    @Override
    public boolean hasNext() {
        return !stack.isEmpty();
    }
    
    @Override
    public Integer next() {
        if (!hasNext()) return null;
        
        TreeNode node = stack.pop();
        int result = node.value;
        
        if (node.right != null) {
            pushLeft(node.right);
        }
        
        return result;
    }
}

// Pre-order Iterator (Root, Left, Right)
class PreOrderIterator implements TreeIterator {
    private Stack<TreeNode> stack = new Stack<>();
    
    public PreOrderIterator(TreeNode root) {
        if (root != null) {
            stack.push(root);
        }
    }
    
    @Override
    public boolean hasNext() {
        return !stack.isEmpty();
    }
    
    @Override
    public Integer next() {
        if (!hasNext()) return null;
        
        TreeNode node = stack.pop();
        
        // Push right first so left is processed first
        if (node.right != null) stack.push(node.right);
        if (node.left != null) stack.push(node.left);
        
        return node.value;
    }
}

// Level-order Iterator (BFS)
class LevelOrderIterator implements TreeIterator {
    private Queue<TreeNode> queue = new LinkedList<>();
    
    public LevelOrderIterator(TreeNode root) {
        if (root != null) {
            queue.add(root);
        }
    }
    
    @Override
    public boolean hasNext() {
        return !queue.isEmpty();
    }
    
    @Override
    public Integer next() {
        if (!hasNext()) return null;
        
        TreeNode node = queue.poll();
        
        if (node.left != null) queue.add(node.left);
        if (node.right != null) queue.add(node.right);
        
        return node.value;
    }
}

// Binary Tree with multiple iterators
class BinaryTree {
    private TreeNode root;
    
    public BinaryTree(TreeNode root) {
        this.root = root;
    }
    
    public TreeIterator inOrderIterator() {
        return new InOrderIterator(root);
    }
    
    public TreeIterator preOrderIterator() {
        return new PreOrderIterator(root);
    }
    
    public TreeIterator levelOrderIterator() {
        return new LevelOrderIterator(root);
    }
}

// Usage
public class TreeTraversal {
    public static void main(String[] args) {
        /*
                 4
               /   \
              2     6
             / \   / \
            1   3 5   7
        */
        TreeNode root = new TreeNode(4);
        root.left = new TreeNode(2);
        root.right = new TreeNode(6);
        root.left.left = new TreeNode(1);
        root.left.right = new TreeNode(3);
        root.right.left = new TreeNode(5);
        root.right.right = new TreeNode(7);
        
        BinaryTree tree = new BinaryTree(root);
        
        // In-order: 1, 2, 3, 4, 5, 6, 7
        System.out.print("In-order: ");
        TreeIterator inOrder = tree.inOrderIterator();
        while (inOrder.hasNext()) {
            System.out.print(inOrder.next() + " ");
        }
        
        // Pre-order: 4, 2, 1, 3, 6, 5, 7
        System.out.print("\nPre-order: ");
        TreeIterator preOrder = tree.preOrderIterator();
        while (preOrder.hasNext()) {
            System.out.print(preOrder.next() + " ");
        }
        
        // Level-order: 4, 2, 6, 1, 3, 5, 7
        System.out.print("\nLevel-order: ");
        TreeIterator levelOrder = tree.levelOrderIterator();
        while (levelOrder.hasNext()) {
            System.out.print(levelOrder.next() + " ");
        }
    }
}
```

---

### Example 3: Menu Iterator

```java
// Menu Item
class MenuItem {
    private String name;
    private String description;
    private double price;
    private boolean vegetarian;
    
    public MenuItem(String name, String description, double price, boolean vegetarian) {
        this.name = name;
        this.description = description;
        this.price = price;
        this.vegetarian = vegetarian;
    }
    
    public String getName() { return name; }
    public String getDescription() { return description; }
    public double getPrice() { return price; }
    public boolean isVegetarian() { return vegetarian; }
}

// Menu interface with iterator
interface Menu {
    java.util.Iterator<MenuItem> createIterator();
}

// Breakfast Menu (uses ArrayList)
class BreakfastMenu implements Menu {
    private List<MenuItem> menuItems;
    
    public BreakfastMenu() {
        menuItems = new ArrayList<>();
        addItem("Pancakes", "Fluffy pancakes with maple syrup", 4.99, true);
        addItem("Eggs & Bacon", "Two eggs with crispy bacon", 5.99, false);
        addItem("Oatmeal", "Steel-cut oats with berries", 3.99, true);
    }
    
    public void addItem(String name, String desc, double price, boolean veg) {
        menuItems.add(new MenuItem(name, desc, price, veg));
    }
    
    @Override
    public java.util.Iterator<MenuItem> createIterator() {
        return menuItems.iterator();  // ArrayList provides iterator
    }
}

// Dinner Menu (uses Array)
class DinnerMenu implements Menu {
    private MenuItem[] menuItems;
    private int itemCount = 0;
    private static final int MAX_ITEMS = 10;
    
    public DinnerMenu() {
        menuItems = new MenuItem[MAX_ITEMS];
        addItem("Steak", "Grilled ribeye with vegetables", 24.99, false);
        addItem("Salmon", "Atlantic salmon with rice", 18.99, false);
        addItem("Pasta", "Spaghetti with marinara sauce", 12.99, true);
    }
    
    public void addItem(String name, String desc, double price, boolean veg) {
        if (itemCount < MAX_ITEMS) {
            menuItems[itemCount++] = new MenuItem(name, desc, price, veg);
        }
    }
    
    @Override
    public java.util.Iterator<MenuItem> createIterator() {
        return new DinnerMenuIterator(menuItems, itemCount);
    }
}

// Custom iterator for array-based menu
class DinnerMenuIterator implements java.util.Iterator<MenuItem> {
    private MenuItem[] items;
    private int position = 0;
    private int size;
    
    public DinnerMenuIterator(MenuItem[] items, int size) {
        this.items = items;
        this.size = size;
    }
    
    @Override
    public boolean hasNext() {
        return position < size && items[position] != null;
    }
    
    @Override
    public MenuItem next() {
        return items[position++];
    }
}

// Waitress uses iterator - doesn't care about menu implementation!
class Waitress {
    private List<Menu> menus;
    
    public Waitress(List<Menu> menus) {
        this.menus = menus;
    }
    
    public void printMenu() {
        for (Menu menu : menus) {
            java.util.Iterator<MenuItem> iterator = menu.createIterator();
            printMenu(iterator);
            System.out.println();
        }
    }
    
    private void printMenu(java.util.Iterator<MenuItem> iterator) {
        while (iterator.hasNext()) {
            MenuItem item = iterator.next();
            System.out.printf("%-20s $%.2f%n", item.getName(), item.getPrice());
            System.out.println("  " + item.getDescription());
            if (item.isVegetarian()) {
                System.out.println("  (V) Vegetarian");
            }
        }
    }
    
    public void printVegetarianMenu() {
        System.out.println("=== Vegetarian Options ===\n");
        for (Menu menu : menus) {
            java.util.Iterator<MenuItem> iterator = menu.createIterator();
            while (iterator.hasNext()) {
                MenuItem item = iterator.next();
                if (item.isVegetarian()) {
                    System.out.printf("%-20s $%.2f%n", item.getName(), item.getPrice());
                }
            }
        }
    }
}

// Usage
public class Restaurant {
    public static void main(String[] args) {
        List<Menu> menus = Arrays.asList(
            new BreakfastMenu(),
            new DinnerMenu()
        );
        
        Waitress waitress = new Waitress(menus);
        
        System.out.println("=== FULL MENU ===\n");
        waitress.printMenu();
        
        waitress.printVegetarianMenu();
    }
}
```

---

## Java's Iterator Interface

```java
// Java provides Iterator in java.util
public interface Iterator<E> {
    boolean hasNext();
    E next();
    default void remove() {
        throw new UnsupportedOperationException("remove");
    }
}

// And Iterable interface
public interface Iterable<T> {
    Iterator<T> iterator();
}

// Any class implementing Iterable can use for-each!
class CustomList<T> implements Iterable<T> {
    private List<T> items = new ArrayList<>();
    
    public void add(T item) { items.add(item); }
    
    @Override
    public Iterator<T> iterator() {
        return items.iterator();
    }
}

// Usage with for-each loop
CustomList<String> list = new CustomList<>();
list.add("A");
list.add("B");

for (String s : list) {  // Works because of Iterable!
    System.out.println(s);
}
```

---

## When to Use Iterator Pattern

### ✅ Use When:
1. Access elements **without exposing** internal structure
2. Support **multiple traversals** of same collection
3. Provide **uniform interface** for different collections
4. Hide complexity of traversing complex structures (trees, graphs)

### ❌ Don't Use When:
1. Collection is simple and traversal is trivial
2. Only one traversal method is ever needed
3. Direct access is more efficient

---

## Iterator vs Other Patterns

| Pattern | Comparison |
|---------|------------|
| **Iterator** | Navigate collection elements |
| **Visitor** | Perform operation on each element |
| **Composite** | Often uses Iterator for traversal |

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Intent** | Sequential access without exposing representation |
| **Key Idea** | Separate traversal logic from collection |
| **Benefits** | Uniform interface, multiple iterators, SRP |
| **Use Case** | Collections, trees, graphs |

### Remember:
- Iterator **encapsulates** traversal logic
- Collection provides **create iterator** method
- Client uses **hasNext()** and **next()**
- Same collection can have **multiple iterators** (different traversals)

---

**Next: Template Method Pattern →**
