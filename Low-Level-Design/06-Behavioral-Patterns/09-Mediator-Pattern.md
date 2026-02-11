# Mediator Design Pattern

## Intent

> **Define an object that encapsulates how a set of objects interact. Mediator promotes loose coupling by keeping objects from referring to each other explicitly, and it lets you vary their interaction independently.**

---

## The Problem

You have **multiple objects** that need to **communicate with each other**:
- Each object knows about all others → **tight coupling**
- Adding new object = modifying all existing ones
- Complex **web of dependencies**
- **Spaghetti** communication

```java
// Without Mediator - Everyone knows everyone! ❌
class Button {
    private TextBox textBox;
    private ListBox listBox;
    private CheckBox checkBox;
    // Too many dependencies!
    
    void click() {
        textBox.clear();
        listBox.refresh();
        checkBox.uncheck();
    }
}
```

---

## Simple Analogy

Think of an **Air Traffic Controller**:
- Planes don't talk directly to each other
- All communication goes through ATC (mediator)
- ATC coordinates takeoffs, landings, prevents collisions
- Planes only interact with ONE entity (ATC), not hundreds

Or think of a **Chat Room**:
- Users don't send messages directly to each other
- Messages go through chat room (mediator)
- Room handles who receives what message

---

## Structure

```
WITHOUT MEDIATOR:                 WITH MEDIATOR:
                                  
  ┌───┐ ───── ┌───┐                    ┌───────────┐
  │ A │ ───── │ B │                    │ Mediator  │
  └───┘ ───── └───┘                    └─────┬─────┘
    │ ╲     ╱ │                        ╱     │     ╲
    │   ╲ ╱   │                      ╱       │       ╲
    │   ╱ ╲   │                  ┌───┐    ┌───┐    ┌───┐
    │ ╱     ╲ │                  │ A │    │ B │    │ C │
  ┌───┐ ───── ┌───┐              └───┘    └───┘    └───┘
  │ D │ ───── │ C │              
  └───┘       └───┘              Each component only knows mediator!
                                  
  Everyone knows everyone!        Components don't know each other!
```

```
┌────────────────────────────────────────────────────────────────────────┐
│                      «interface» Mediator                               │
├────────────────────────────────────────────────────────────────────────┤
│ + notify(sender: Component, event: String)                              │
└──────────────────────────────────────△─────────────────────────────────┘
                                       │
                                       │
                         ┌─────────────────────────────┐
                         │      ConcreteMediator       │
                         ├─────────────────────────────┤
                         │ - componentA: ComponentA    │
                         │ - componentB: ComponentB    │
                         ├─────────────────────────────┤
                         │ + notify(sender, event) {   │
                         │     // Coordinate           │
                         │   }                         │
                         └─────────────────────────────┘
                              ╱                ╲
                            ╱                    ╲
              ┌────────────────────┐   ┌────────────────────┐
              │    ComponentA      │   │    ComponentB      │
              ├────────────────────┤   ├────────────────────┤
              │ - mediator         │   │ - mediator         │
              ├────────────────────┤   ├────────────────────┤
              │ + operation() {    │   │ + operation() {    │
              │   mediator.notify  │   │   mediator.notify  │
              │     (this,"event") │   │     (this,"event") │
              │   }                │   │   }                │
              └────────────────────┘   └────────────────────┘
```

---

## Basic Example: Chat Room

```java
// Mediator interface
interface ChatMediator {
    void sendMessage(String message, User sender);
    void addUser(User user);
}

// Colleague (Component) base
abstract class User {
    protected ChatMediator mediator;
    protected String name;
    
    public User(ChatMediator mediator, String name) {
        this.mediator = mediator;
        this.name = name;
    }
    
    public String getName() { return name; }
    
    public abstract void send(String message);
    public abstract void receive(String message, String fromUser);
}

// Concrete Mediator
class ChatRoom implements ChatMediator {
    private List<User> users = new ArrayList<>();
    
    @Override
    public void addUser(User user) {
        users.add(user);
        System.out.println(user.getName() + " joined the chat");
    }
    
    @Override
    public void sendMessage(String message, User sender) {
        for (User user : users) {
            // Don't send to sender
            if (user != sender) {
                user.receive(message, sender.getName());
            }
        }
    }
}

// Concrete Colleague
class ChatUser extends User {
    public ChatUser(ChatMediator mediator, String name) {
        super(mediator, name);
    }
    
    @Override
    public void send(String message) {
        System.out.println(name + " sends: " + message);
        mediator.sendMessage(message, this);
    }
    
    @Override
    public void receive(String message, String fromUser) {
        System.out.println(name + " received from " + fromUser + ": " + message);
    }
}

// Usage
public class ChatDemo {
    public static void main(String[] args) {
        ChatMediator chatRoom = new ChatRoom();
        
        User john = new ChatUser(chatRoom, "John");
        User jane = new ChatUser(chatRoom, "Jane");
        User bob = new ChatUser(chatRoom, "Bob");
        
        chatRoom.addUser(john);
        chatRoom.addUser(jane);
        chatRoom.addUser(bob);
        
        System.out.println();
        john.send("Hello everyone!");
        
        System.out.println();
        jane.send("Hi John!");
    }
}

// Output:
// John joined the chat
// Jane joined the chat
// Bob joined the chat
// 
// John sends: Hello everyone!
// Jane received from John: Hello everyone!
// Bob received from John: Hello everyone!
// 
// Jane sends: Hi John!
// John received from Jane: Hi John!
// Bob received from Jane: Hi John!
```

---

## Real-World Examples

### Example 1: UI Dialog Mediator

```java
// Component base
abstract class UIComponent {
    protected Dialog dialog;
    protected String name;
    
    public UIComponent(String name) {
        this.name = name;
    }
    
    public void setMediator(Dialog dialog) {
        this.dialog = dialog;
    }
    
    public String getName() { return name; }
    
    protected void changed() {
        if (dialog != null) {
            dialog.componentChanged(this);
        }
    }
}

// Concrete Components
class TextBox extends UIComponent {
    private String text = "";
    
    public TextBox(String name) {
        super(name);
    }
    
    public void setText(String text) {
        this.text = text;
        System.out.println(name + " text changed to: " + text);
        changed();
    }
    
    public String getText() { return text; }
    
    public void clear() {
        this.text = "";
        System.out.println(name + " cleared");
    }
}

class CheckBox extends UIComponent {
    private boolean checked = false;
    
    public CheckBox(String name) {
        super(name);
    }
    
    public void check() {
        this.checked = true;
        System.out.println(name + " checked");
        changed();
    }
    
    public void uncheck() {
        this.checked = false;
        System.out.println(name + " unchecked");
        changed();
    }
    
    public boolean isChecked() { return checked; }
}

class Button extends UIComponent {
    private boolean enabled = false;
    
    public Button(String name) {
        super(name);
    }
    
    public void click() {
        if (enabled) {
            System.out.println(name + " clicked");
            changed();
        } else {
            System.out.println(name + " is disabled - cannot click");
        }
    }
    
    public void enable() {
        this.enabled = true;
        System.out.println(name + " enabled");
    }
    
    public void disable() {
        this.enabled = false;
        System.out.println(name + " disabled");
    }
    
    public boolean isEnabled() { return enabled; }
}

class ListBox extends UIComponent {
    private List<String> items = new ArrayList<>();
    private String selected;
    
    public ListBox(String name) {
        super(name);
    }
    
    public void addItem(String item) {
        items.add(item);
    }
    
    public void select(String item) {
        if (items.contains(item)) {
            selected = item;
            System.out.println(name + " selected: " + item);
            changed();
        }
    }
    
    public String getSelected() { return selected; }
}

// Mediator interface
interface DialogMediator {
    void componentChanged(UIComponent component);
}

// Concrete Mediator
class LoginDialog implements DialogMediator {
    private TextBox usernameInput;
    private TextBox passwordInput;
    private CheckBox rememberMe;
    private Button loginButton;
    private Button cancelButton;
    
    public void setComponents(TextBox username, TextBox password, 
                              CheckBox remember, Button login, Button cancel) {
        this.usernameInput = username;
        this.passwordInput = password;
        this.rememberMe = remember;
        this.loginButton = login;
        this.cancelButton = cancel;
        
        // Register mediator with components
        username.setMediator(this);
        password.setMediator(this);
        remember.setMediator(this);
        login.setMediator(this);
        cancel.setMediator(this);
        
        // Initial state
        loginButton.disable();
        cancelButton.enable();
    }
    
    @Override
    public void componentChanged(UIComponent component) {
        System.out.println("  [Mediator] Processing change from: " + component.getName());
        
        if (component == usernameInput || component == passwordInput) {
            // Enable login button only if both fields have text
            boolean hasUsername = !usernameInput.getText().isEmpty();
            boolean hasPassword = !passwordInput.getText().isEmpty();
            
            if (hasUsername && hasPassword) {
                loginButton.enable();
            } else {
                loginButton.disable();
            }
        } else if (component == loginButton) {
            // Handle login
            System.out.println("  [Mediator] Attempting login for: " + usernameInput.getText());
            if (rememberMe.isChecked()) {
                System.out.println("  [Mediator] Will remember user");
            }
        } else if (component == cancelButton) {
            // Handle cancel
            System.out.println("  [Mediator] Cancelling - clearing fields");
            usernameInput.clear();
            passwordInput.clear();
            rememberMe.uncheck();
        }
    }
}

// Usage
public class DialogDemo {
    public static void main(String[] args) {
        LoginDialog dialog = new LoginDialog();
        
        TextBox username = new TextBox("Username");
        TextBox password = new TextBox("Password");
        CheckBox rememberMe = new CheckBox("RememberMe");
        Button loginBtn = new Button("LoginButton");
        Button cancelBtn = new Button("CancelButton");
        
        dialog.setComponents(username, password, rememberMe, loginBtn, cancelBtn);
        
        System.out.println("\n=== User types username ===");
        username.setText("john");
        
        System.out.println("\n=== User types password ===");
        password.setText("secret123");
        
        System.out.println("\n=== User checks Remember Me ===");
        rememberMe.check();
        
        System.out.println("\n=== User clicks Login ===");
        loginBtn.click();
    }
}
```

---

### Example 2: Smart Home Mediator

```java
// Smart Device base
abstract class SmartDevice {
    protected SmartHomeHub hub;
    protected String name;
    protected boolean on = false;
    
    public SmartDevice(String name) {
        this.name = name;
    }
    
    public void setHub(SmartHomeHub hub) {
        this.hub = hub;
    }
    
    public String getName() { return name; }
    public boolean isOn() { return on; }
    
    protected void notifyHub(String event) {
        if (hub != null) {
            hub.onDeviceEvent(this, event);
        }
    }
}

// Concrete Devices
class Light extends SmartDevice {
    private int brightness = 100;
    
    public Light(String name) {
        super(name);
    }
    
    public void turnOn() {
        on = true;
        System.out.println("💡 " + name + " turned ON");
        notifyHub("LIGHT_ON");
    }
    
    public void turnOff() {
        on = false;
        System.out.println("💡 " + name + " turned OFF");
        notifyHub("LIGHT_OFF");
    }
    
    public void dim(int level) {
        brightness = level;
        System.out.println("💡 " + name + " dimmed to " + level + "%");
    }
}

class Thermostat extends SmartDevice {
    private int temperature = 72;
    
    public Thermostat(String name) {
        super(name);
    }
    
    public void setTemperature(int temp) {
        this.temperature = temp;
        System.out.println("🌡️ " + name + " set to " + temp + "°F");
        notifyHub("TEMP_CHANGED");
    }
    
    public int getTemperature() { return temperature; }
}

class SecurityCamera extends SmartDevice {
    public SecurityCamera(String name) {
        super(name);
    }
    
    public void activate() {
        on = true;
        System.out.println("📹 " + name + " activated - Recording");
        notifyHub("CAMERA_ACTIVE");
    }
    
    public void deactivate() {
        on = false;
        System.out.println("📹 " + name + " deactivated");
        notifyHub("CAMERA_INACTIVE");
    }
    
    public void detectMotion() {
        System.out.println("📹 " + name + " detected motion!");
        notifyHub("MOTION_DETECTED");
    }
}

class DoorLock extends SmartDevice {
    private boolean locked = true;
    
    public DoorLock(String name) {
        super(name);
    }
    
    public void lock() {
        locked = true;
        System.out.println("🔒 " + name + " locked");
        notifyHub("DOOR_LOCKED");
    }
    
    public void unlock() {
        locked = false;
        System.out.println("🔓 " + name + " unlocked");
        notifyHub("DOOR_UNLOCKED");
    }
    
    public boolean isLocked() { return locked; }
}

class Alarm extends SmartDevice {
    public Alarm(String name) {
        super(name);
    }
    
    public void arm() {
        on = true;
        System.out.println("🚨 " + name + " armed");
    }
    
    public void disarm() {
        on = false;
        System.out.println("🚨 " + name + " disarmed");
    }
    
    public void trigger() {
        if (on) {
            System.out.println("🚨🚨🚨 " + name + " TRIGGERED! ALERT! 🚨🚨🚨");
        }
    }
}

// Mediator
class SmartHomeHub {
    private List<Light> lights = new ArrayList<>();
    private Thermostat thermostat;
    private List<SecurityCamera> cameras = new ArrayList<>();
    private DoorLock doorLock;
    private Alarm alarm;
    private boolean awayMode = false;
    
    public void registerLight(Light light) {
        light.setHub(this);
        lights.add(light);
    }
    
    public void registerThermostat(Thermostat t) {
        t.setHub(this);
        thermostat = t;
    }
    
    public void registerCamera(SecurityCamera cam) {
        cam.setHub(this);
        cameras.add(cam);
    }
    
    public void registerDoorLock(DoorLock lock) {
        lock.setHub(this);
        doorLock = lock;
    }
    
    public void registerAlarm(Alarm a) {
        a.setHub(this);
        alarm = a;
    }
    
    // Mediator logic
    public void onDeviceEvent(SmartDevice device, String event) {
        System.out.println("  [Hub] Received: " + event + " from " + device.getName());
        
        switch (event) {
            case "MOTION_DETECTED":
                handleMotionDetected();
                break;
            case "DOOR_UNLOCKED":
                handleDoorUnlocked();
                break;
            case "DOOR_LOCKED":
                handleDoorLocked();
                break;
        }
    }
    
    private void handleMotionDetected() {
        if (awayMode) {
            System.out.println("  [Hub] Away mode - Triggering security response!");
            alarm.trigger();
            for (Light light : lights) {
                light.turnOn();
            }
        }
    }
    
    private void handleDoorUnlocked() {
        // Welcome home routine
        if (awayMode) {
            System.out.println("  [Hub] Welcome home!");
            awayMode = false;
            alarm.disarm();
            for (Light light : lights) {
                light.turnOn();
            }
            thermostat.setTemperature(72);
        }
    }
    
    private void handleDoorLocked() {
        // Check if everyone left
        // For demo, just dim lights
        for (Light light : lights) {
            light.dim(20);
        }
    }
    
    // Scenes
    public void setAwayMode() {
        System.out.println("\n🏠 [Hub] Setting AWAY MODE");
        awayMode = true;
        
        for (Light light : lights) {
            light.turnOff();
        }
        thermostat.setTemperature(65);  // Energy saving
        for (SecurityCamera cam : cameras) {
            cam.activate();
        }
        doorLock.lock();
        alarm.arm();
    }
    
    public void setHomeMode() {
        System.out.println("\n🏠 [Hub] Setting HOME MODE");
        awayMode = false;
        
        for (Light light : lights) {
            light.turnOn();
        }
        thermostat.setTemperature(72);
        alarm.disarm();
    }
}

// Usage
public class SmartHomeDemo {
    public static void main(String[] args) {
        SmartHomeHub hub = new SmartHomeHub();
        
        Light livingRoom = new Light("Living Room Light");
        Light bedroom = new Light("Bedroom Light");
        Thermostat thermo = new Thermostat("Main Thermostat");
        SecurityCamera frontDoor = new SecurityCamera("Front Door Cam");
        DoorLock mainDoor = new DoorLock("Main Door");
        Alarm homeAlarm = new Alarm("Home Alarm");
        
        hub.registerLight(livingRoom);
        hub.registerLight(bedroom);
        hub.registerThermostat(thermo);
        hub.registerCamera(frontDoor);
        hub.registerDoorLock(mainDoor);
        hub.registerAlarm(homeAlarm);
        
        // Leaving home
        hub.setAwayMode();
        
        // Motion while away!
        System.out.println("\n=== Motion Detected! ===");
        frontDoor.detectMotion();
        
        // Coming home
        System.out.println("\n=== Owner Returns ===");
        mainDoor.unlock();
    }
}
```

---

### Example 3: Airline Flight Coordination

```java
// Base
abstract class Aircraft {
    protected AirTrafficControl atc;
    protected String callSign;
    protected int altitude;
    
    public Aircraft(String callSign) {
        this.callSign = callSign;
    }
    
    public void setATC(AirTrafficControl atc) {
        this.atc = atc;
    }
    
    public String getCallSign() { return callSign; }
    public int getAltitude() { return altitude; }
    
    public abstract void requestLanding();
    public abstract void requestTakeoff();
}

class CommercialFlight extends Aircraft {
    public CommercialFlight(String callSign) {
        super(callSign);
        this.altitude = 35000;
    }
    
    @Override
    public void requestLanding() {
        System.out.println("✈️ " + callSign + ": Requesting landing clearance");
        atc.requestLanding(this);
    }
    
    @Override
    public void requestTakeoff() {
        System.out.println("✈️ " + callSign + ": Requesting takeoff clearance");
        atc.requestTakeoff(this);
    }
    
    public void land() {
        altitude = 0;
        System.out.println("✈️ " + callSign + ": Landed safely");
    }
    
    public void takeoff() {
        altitude = 35000;
        System.out.println("✈️ " + callSign + ": Airborne, climbing to cruise altitude");
    }
    
    public void holdPattern() {
        System.out.println("✈️ " + callSign + ": Entering holding pattern");
    }
}

// Mediator
interface AirTrafficControl {
    void registerAircraft(Aircraft aircraft);
    void requestLanding(Aircraft aircraft);
    void requestTakeoff(Aircraft aircraft);
}

class RunwayATC implements AirTrafficControl {
    private List<Aircraft> airborne = new ArrayList<>();
    private List<Aircraft> grounded = new ArrayList<>();
    private boolean runwayFree = true;
    private Queue<Aircraft> landingQueue = new LinkedList<>();
    private Queue<Aircraft> takeoffQueue = new LinkedList<>();
    
    @Override
    public void registerAircraft(Aircraft aircraft) {
        aircraft.setATC(this);
        if (aircraft.getAltitude() > 0) {
            airborne.add(aircraft);
        } else {
            grounded.add(aircraft);
        }
    }
    
    @Override
    public void requestLanding(Aircraft aircraft) {
        if (runwayFree && landingQueue.isEmpty() && takeoffQueue.isEmpty()) {
            clearForLanding(aircraft);
        } else {
            System.out.println("🗼 ATC: " + aircraft.getCallSign() 
                + ", hold position, you are #" + (landingQueue.size() + 1) + " in queue");
            landingQueue.offer(aircraft);
            if (aircraft instanceof CommercialFlight) {
                ((CommercialFlight) aircraft).holdPattern();
            }
        }
    }
    
    @Override
    public void requestTakeoff(Aircraft aircraft) {
        if (runwayFree && landingQueue.isEmpty() && takeoffQueue.isEmpty()) {
            clearForTakeoff(aircraft);
        } else {
            System.out.println("🗼 ATC: " + aircraft.getCallSign() 
                + ", hold position, you are #" + (takeoffQueue.size() + 1) + " in queue");
            takeoffQueue.offer(aircraft);
        }
    }
    
    private void clearForLanding(Aircraft aircraft) {
        runwayFree = false;
        System.out.println("🗼 ATC: " + aircraft.getCallSign() + ", cleared for landing runway 27L");
        if (aircraft instanceof CommercialFlight) {
            ((CommercialFlight) aircraft).land();
        }
        airborne.remove(aircraft);
        grounded.add(aircraft);
        runwayFree = true;
        processQueue();
    }
    
    private void clearForTakeoff(Aircraft aircraft) {
        runwayFree = false;
        System.out.println("🗼 ATC: " + aircraft.getCallSign() + ", cleared for takeoff runway 27L");
        if (aircraft instanceof CommercialFlight) {
            ((CommercialFlight) aircraft).takeoff();
        }
        grounded.remove(aircraft);
        airborne.add(aircraft);
        runwayFree = true;
        processQueue();
    }
    
    private void processQueue() {
        // Prioritize landings (aircraft have limited fuel)
        if (!landingQueue.isEmpty()) {
            clearForLanding(landingQueue.poll());
        } else if (!takeoffQueue.isEmpty()) {
            clearForTakeoff(takeoffQueue.poll());
        }
    }
    
    public void status() {
        System.out.println("\n🗼 ATC STATUS:");
        System.out.println("  Runway: " + (runwayFree ? "FREE" : "OCCUPIED"));
        System.out.println("  Airborne: " + airborne.stream()
            .map(Aircraft::getCallSign).toList());
        System.out.println("  Grounded: " + grounded.stream()
            .map(Aircraft::getCallSign).toList());
    }
}

// Usage
public class ATCDemo {
    public static void main(String[] args) {
        RunwayATC atc = new RunwayATC();
        
        CommercialFlight ua123 = new CommercialFlight("UA123");
        CommercialFlight aa456 = new CommercialFlight("AA456");
        CommercialFlight dl789 = new CommercialFlight("DL789");
        
        // DL789 is on ground
        dl789.altitude = 0;
        
        atc.registerAircraft(ua123);
        atc.registerAircraft(aa456);
        atc.registerAircraft(dl789);
        
        atc.status();
        
        // Multiple requests
        System.out.println("\n=== Flight Requests ===");
        ua123.requestLanding();
        aa456.requestLanding();
        dl789.requestTakeoff();
        
        atc.status();
    }
}
```

---

## When to Use

### ✅ Use When:
1. Objects have **many-to-many** relationships
2. Communication is **complex** between objects
3. You want to **centralize** control logic
4. Objects should be **reusable** independently

### ❌ Don't Use When:
1. Only **2-3 objects** communicating
2. Mediator would become **God object**
3. Communication is already **simple**

---

## Mediator vs Observer

| Mediator | Observer |
|----------|----------|
| **Centralized** communication | **Distributed** notification |
| Objects know mediator | Subjects know observers |
| Two-way communication | One-way notification |
| **Controls** interaction | Just **notifies** |

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Intent** | Centralize complex communication |
| **Key Idea** | Objects talk through mediator, not directly |
| **Benefits** | Decoupling, SRP, easier to add components |
| **Drawback** | Mediator can become God object |
| **Use Cases** | UI dialogs, chat systems, smart home, ATC |

### Remember:
- All components know **only mediator**
- Mediator knows **all components**
- **Decouples** many-to-many relationships
- Watch out for **God object** antipattern!

---

**Next: Memento Pattern →**
