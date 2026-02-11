# LLD Interview Approach

## The Framework: SOLID-D-E-C

Use this framework for every LLD interview:

1. **S** - Scope: Clarify requirements
2. **O** - Objects: Identify entities and classes
3. **L** - Layout: Design class relationships (UML)
4. **I** - Interfaces: Define APIs and contracts
5. **D** - Data: Choose data structures
6. **D** - Design Patterns: Apply appropriate patterns
7. **E** - Edge Cases: Handle exceptions
8. **C** - Code: Write clean implementation

---

## Step 1: Clarify Requirements (5-10 min)

### Ask These Questions

```
FUNCTIONAL REQUIREMENTS:
- What are the core features?
- Who are the actors/users?
- What operations should be supported?
- What's in scope vs out of scope?

NON-FUNCTIONAL REQUIREMENTS:
- Scale: How many users/requests?
- Concurrency: Multi-threading needed?
- Persistence: Database or in-memory?
- Extensibility: Future features to consider?

Example for Parking Lot:
Q: "What types of vehicles?"
A: "Bike, Car, Truck"

Q: "Multiple floors?"
A: "Yes, assume multiple floors"

Q: "Payment required?"
A: "Yes, calculate based on duration"

Q: "Concurrency?"
A: "Yes, multiple entry/exit points"
```

### Document Requirements

```
=== PARKING LOT SYSTEM ===

ACTORS:
- Customer
- Admin

CORE FEATURES:
1. Park vehicle
2. Unpark vehicle
3. Find available spot
4. Calculate parking fee
5. Multiple floors support

CONSTRAINTS:
- Spots: Bike < Car < Truck (size)
- Pricing: Hourly rate per vehicle type
- Concurrent access handling

OUT OF SCOPE:
- Reservations
- Electric vehicle charging
```

---

## Step 2: Identify Entities (5 min)

### Extract Nouns from Requirements

```
From "A customer parks their car in a parking spot on floor 2":
- Customer ✗ (external actor, may not need class)
- Car → Vehicle (generalize)
- Parking Spot → ParkingSpot
- Floor → ParkingFloor
- Parking Lot → ParkingLot

From "Calculate fee based on hourly rate per vehicle type":
- Fee → ParkingFee or PaymentService
- Hourly Rate → part of pricing strategy
- Vehicle Type → VehicleType enum
```

### Entity List

```java
// Entities
- ParkingLot
- ParkingFloor
- ParkingSpot
- Vehicle (abstract)
  - Bike
  - Car
  - Truck
- Ticket
- Payment

// Enums
- VehicleType
- SpotType
- SpotStatus
- PaymentStatus

// Services
- ParkingService
- PaymentService
```

---

## Step 3: Design Class Relationships (10 min)

### Draw Simple UML

```
┌─────────────┐       ┌──────────────────┐
│ ParkingLot  │──────<│   ParkingFloor   │
├─────────────┤  1:N  ├──────────────────┤
│ name        │       │ floorNumber      │
│ floors[]    │       │ spots[]          │
├─────────────┤       ├──────────────────┤
│ findSpot()  │       │ getAvailableSpot│
│ parkVehicle │       │                  │
└─────────────┘       └────────┬─────────┘
                               │ 1:N
                      ┌────────┴─────────┐
                      │   ParkingSpot    │
                      ├──────────────────┤
                      │ spotId           │
                      │ spotType         │
                      │ status           │
                      │ vehicle          │
                      ├──────────────────┤
                      │ isAvailable()    │
                      │ park(vehicle)    │
                      │ unpark()         │
                      └──────────────────┘

┌──────────────┐      ┌───────────────┐
│   Vehicle    │◄─────│    Ticket     │
├──────────────┤      ├───────────────┤
│ licensePlate │      │ ticketId      │
│ vehicleType  │      │ entryTime     │
├──────────────┤      │ exitTime      │
│              │      │ spot          │
└──────────────┘      │ vehicle       │
       △              ├───────────────┤
       │              │ calculateFee()│
  ┌────┴────┐         └───────────────┘
  │    │    │
Bike  Car  Truck
```

---

## Step 4: Define Interfaces (5 min)

### Core APIs

```java
// Main operations
public interface ParkingLotService {
    Ticket parkVehicle(Vehicle vehicle);
    Payment unparkVehicle(String ticketId);
    int getAvailableSpotsCount(VehicleType type);
}

// Spot finding strategy
public interface SpotFindingStrategy {
    ParkingSpot findSpot(List<ParkingFloor> floors, VehicleType type);
}

// Pricing strategy
public interface PricingStrategy {
    double calculateFee(Ticket ticket);
}
```

---

## Step 5: Choose Data Structures (5 min)

```java
// For O(1) lookups
Map<String, Ticket> activeTickets;        // ticketId → Ticket
Map<String, ParkingSpot> vehicleToSpot;   // licensePlate → Spot

// For finding available spots
Queue<ParkingSpot> bikeSpots;
Queue<ParkingSpot> carSpots;
Queue<ParkingSpot> truckSpots;

// For capacity tracking
AtomicInteger availableBikeSpots;
AtomicInteger availableCarSpots;
AtomicInteger availableTruckSpots;
```

---

## Step 6: Apply Design Patterns (5 min)

### Pattern Selection

```java
// SINGLETON - Single parking lot instance
public class ParkingLot {
    private static ParkingLot instance;
    
    public static ParkingLot getInstance() {
        if (instance == null) {
            synchronized (ParkingLot.class) {
                if (instance == null) {
                    instance = new ParkingLot();
                }
            }
        }
        return instance;
    }
}

// FACTORY - Create vehicles
public class VehicleFactory {
    public static Vehicle createVehicle(VehicleType type, String licensePlate) {
        switch (type) {
            case BIKE: return new Bike(licensePlate);
            case CAR: return new Car(licensePlate);
            case TRUCK: return new Truck(licensePlate);
            default: throw new IllegalArgumentException();
        }
    }
}

// STRATEGY - Different pricing models
public interface PricingStrategy {
    double calculateFee(Ticket ticket);
}

public class HourlyPricing implements PricingStrategy {
    private Map<VehicleType, Double> hourlyRates;
    
    public double calculateFee(Ticket ticket) {
        long hours = ChronoUnit.HOURS.between(ticket.getEntryTime(), 
                                              ticket.getExitTime());
        return hourlyRates.get(ticket.getVehicleType()) * Math.max(1, hours);
    }
}

public class WeekendPricing implements PricingStrategy {
    // Different rates for weekends
}
```

---

## Step 7: Handle Edge Cases (5 min)

```java
// List edge cases during design
/*
EDGE CASES:
1. Parking lot full - throw ParkingLotFullException
2. Invalid ticket - throw InvalidTicketException  
3. Vehicle already parked - throw DuplicateVehicleException
4. Concurrent parking at same spot - use synchronization
5. Power failure - persist state
6. Lost ticket - manual verification flow
*/

// Handle concurrency
public class ParkingSpot {
    private final Lock lock = new ReentrantLock();
    private Vehicle vehicle;
    
    public boolean tryPark(Vehicle vehicle) {
        if (lock.tryLock()) {
            try {
                if (isAvailable()) {
                    this.vehicle = vehicle;
                    this.status = SpotStatus.OCCUPIED;
                    return true;
                }
            } finally {
                lock.unlock();
            }
        }
        return false;
    }
}
```

---

## Step 8: Write Clean Code (15-20 min)

### Complete Implementation

```java
// Enums
public enum VehicleType { BIKE, CAR, TRUCK }
public enum SpotType { BIKE, CAR, TRUCK }
public enum SpotStatus { AVAILABLE, OCCUPIED }

// Vehicle hierarchy
public abstract class Vehicle {
    protected String licensePlate;
    protected VehicleType type;
    
    public Vehicle(String licensePlate, VehicleType type) {
        this.licensePlate = licensePlate;
        this.type = type;
    }
    
    public String getLicensePlate() { return licensePlate; }
    public VehicleType getType() { return type; }
}

public class Car extends Vehicle {
    public Car(String licensePlate) {
        super(licensePlate, VehicleType.CAR);
    }
}

// ParkingSpot
public class ParkingSpot {
    private final String spotId;
    private final SpotType spotType;
    private SpotStatus status;
    private Vehicle vehicle;
    
    public ParkingSpot(String spotId, SpotType spotType) {
        this.spotId = spotId;
        this.spotType = spotType;
        this.status = SpotStatus.AVAILABLE;
    }
    
    public synchronized boolean park(Vehicle vehicle) {
        if (!isAvailable()) return false;
        if (!canFit(vehicle)) return false;
        
        this.vehicle = vehicle;
        this.status = SpotStatus.OCCUPIED;
        return true;
    }
    
    public synchronized Vehicle unpark() {
        Vehicle parkedVehicle = this.vehicle;
        this.vehicle = null;
        this.status = SpotStatus.AVAILABLE;
        return parkedVehicle;
    }
    
    public boolean isAvailable() {
        return status == SpotStatus.AVAILABLE;
    }
    
    public boolean canFit(Vehicle vehicle) {
        // Bike fits bike spots, Car fits car spots, etc.
        return spotType.ordinal() >= vehicle.getType().ordinal();
    }
    
    // Getters
}

// Ticket
public class Ticket {
    private final String ticketId;
    private final Vehicle vehicle;
    private final ParkingSpot spot;
    private final LocalDateTime entryTime;
    private LocalDateTime exitTime;
    
    public Ticket(Vehicle vehicle, ParkingSpot spot) {
        this.ticketId = UUID.randomUUID().toString();
        this.vehicle = vehicle;
        this.spot = spot;
        this.entryTime = LocalDateTime.now();
    }
    
    public void markExit() {
        this.exitTime = LocalDateTime.now();
    }
    
    // Getters
}

// ParkingFloor
public class ParkingFloor {
    private final int floorNumber;
    private final Map<SpotType, Queue<ParkingSpot>> availableSpots;
    private final List<ParkingSpot> allSpots;
    
    public ParkingFloor(int floorNumber) {
        this.floorNumber = floorNumber;
        this.availableSpots = new EnumMap<>(SpotType.class);
        this.allSpots = new ArrayList<>();
        
        for (SpotType type : SpotType.values()) {
            availableSpots.put(type, new ConcurrentLinkedQueue<>());
        }
    }
    
    public void addSpot(ParkingSpot spot) {
        allSpots.add(spot);
        availableSpots.get(spot.getSpotType()).offer(spot);
    }
    
    public ParkingSpot getAvailableSpot(VehicleType vehicleType) {
        SpotType requiredSpot = SpotType.valueOf(vehicleType.name());
        return availableSpots.get(requiredSpot).poll();
    }
    
    public void releaseSpot(ParkingSpot spot) {
        availableSpots.get(spot.getSpotType()).offer(spot);
    }
}

// ParkingLot (Singleton)
public class ParkingLot {
    private static volatile ParkingLot instance;
    
    private final List<ParkingFloor> floors;
    private final Map<String, Ticket> activeTickets;
    private final PricingStrategy pricingStrategy;
    
    private ParkingLot() {
        this.floors = new ArrayList<>();
        this.activeTickets = new ConcurrentHashMap<>();
        this.pricingStrategy = new HourlyPricing();
    }
    
    public static ParkingLot getInstance() {
        if (instance == null) {
            synchronized (ParkingLot.class) {
                if (instance == null) {
                    instance = new ParkingLot();
                }
            }
        }
        return instance;
    }
    
    public Ticket parkVehicle(Vehicle vehicle) {
        ParkingSpot spot = findAvailableSpot(vehicle.getType());
        if (spot == null) {
            throw new ParkingLotFullException("No spots available for " + vehicle.getType());
        }
        
        if (!spot.park(vehicle)) {
            throw new RuntimeException("Failed to park vehicle");
        }
        
        Ticket ticket = new Ticket(vehicle, spot);
        activeTickets.put(ticket.getTicketId(), ticket);
        return ticket;
    }
    
    public Payment unparkVehicle(String ticketId) {
        Ticket ticket = activeTickets.remove(ticketId);
        if (ticket == null) {
            throw new InvalidTicketException("Invalid ticket: " + ticketId);
        }
        
        ticket.markExit();
        ParkingSpot spot = ticket.getSpot();
        spot.unpark();
        
        // Return spot to available pool
        int floorNum = extractFloorNumber(spot.getSpotId());
        floors.get(floorNum).releaseSpot(spot);
        
        double fee = pricingStrategy.calculateFee(ticket);
        return new Payment(ticket, fee);
    }
    
    private ParkingSpot findAvailableSpot(VehicleType type) {
        for (ParkingFloor floor : floors) {
            ParkingSpot spot = floor.getAvailableSpot(type);
            if (spot != null) return spot;
        }
        return null;
    }
    
    // Floor management, etc.
}
```

---

## Interview Tips

### DO ✅

| Action | Why |
|--------|-----|
| **Ask clarifying questions** | Shows you don't make assumptions |
| **Think out loud** | Lets interviewer follow your thought process |
| **Start with high-level design** | Shows you can see big picture |
| **Use design patterns** | Shows you know best practices |
| **Handle edge cases** | Shows attention to detail |
| **Write clean, modular code** | Shows coding skills |

### DON'T ❌

| Action | Why |
|--------|-----|
| **Jump into coding immediately** | Misses requirements, looks amateur |
| **Over-engineer** | Wastes time, confuses design |
| **Ignore interviewer hints** | They're trying to help |
| **Write pseudo-code** | Real code shows competence |
| **Forget concurrency** | Critical for real systems |

---

## Common LLD Problems

| Problem | Key Patterns |
|---------|-------------|
| **Parking Lot** | Factory, Strategy, Singleton |
| **BookMyShow** | Observer, Strategy, State |
| **Elevator System** | State, Strategy, Observer |
| **Chess Game** | Factory, Strategy, Command |
| **Tic Tac Toe** | Strategy, State |
| **Vending Machine** | State, Strategy |
| **File System** | Composite, Iterator |
| **Library Management** | Factory, Observer |
| **ATM Machine** | State, Chain of Responsibility |
| **Hotel Booking** | Strategy, Observer, State |

---

## Time Management (45 min)

```
┌──────────────────────────────────────────────────────────────┐
│  0-8 min: Requirements gathering and clarification           │
├──────────────────────────────────────────────────────────────┤
│  8-15 min: Identify entities and relationships              │
├──────────────────────────────────────────────────────────────┤
│ 15-25 min: Design class structure and interfaces            │
├──────────────────────────────────────────────────────────────┤
│ 25-40 min: Write code implementation                        │
├──────────────────────────────────────────────────────────────┤
│ 40-45 min: Review, edge cases, improvements                 │
└──────────────────────────────────────────────────────────────┘
```

---

**Next: Complete Example Solution →**
