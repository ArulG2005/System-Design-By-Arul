# Complete LLD Solution: Parking Lot System

This is a complete, production-ready Low-Level Design solution demonstrating all concepts from this course.

---

## Problem Statement

Design a parking lot system that:
1. Supports multiple floors
2. Has different spot sizes (Bike, Car, Truck)
3. Parks and unparks vehicles
4. Calculates parking fee based on duration
5. Handles concurrent access
6. Tracks available spots per type

---

## Step 1: Requirements

### Functional Requirements
- Park vehicle and get ticket
- Unpark vehicle and pay fee
- Find available spots by vehicle type
- Support multiple vehicle types
- Calculate fee based on hourly rates
- Track entry and exit times

### Non-Functional Requirements
- Thread-safe for concurrent access
- Extensible for new vehicle types
- Clean separation of concerns
- Testable design

---

## Step 2: Class Design (UML)

```
┌─────────────────────────────────────────────────────────────────────┐
│                           PARKING LOT SYSTEM                         │
└─────────────────────────────────────────────────────────────────────┘

                         ┌─────────────────┐
                         │   ParkingLot    │ (Singleton)
                         ├─────────────────┤
                         │ - floors        │
                         │ - activeTickets │
                         │ - pricingStrategy│
                         ├─────────────────┤
                         │ + parkVehicle() │
                         │ + unparkVehicle│
                         │ + getAvailable()│
                         └────────┬────────┘
                                  │ 1:N
                         ┌────────┴────────┐
                         │  ParkingFloor   │
                         ├─────────────────┤
                         │ - floorNumber   │
                         │ - spots         │
                         ├─────────────────┤
                         │ + getSpot()     │
                         │ + releaseSpot() │
                         └────────┬────────┘
                                  │ 1:N
                         ┌────────┴────────┐
                         │   ParkingSpot   │
                         ├─────────────────┤
                         │ - spotId        │
                         │ - spotType      │
                         │ - vehicle       │
                         ├─────────────────┤
                         │ + park()        │
                         │ + unpark()      │
                         │ + isAvailable() │
                         └─────────────────┘

┌─────────────┐         ┌─────────────┐        ┌─────────────────────┐
│   Vehicle   │◄────────│   Ticket    │        │  PricingStrategy    │ (Strategy)
├─────────────┤         ├─────────────┤        ├─────────────────────┤
│ licensePlate│         │ ticketId    │        │ + calculateFee()    │
│ type        │         │ entryTime   │        └──────────┬──────────┘
└──────┬──────┘         │ exitTime    │                   │
       │                │ spot        │        ┌──────────┴──────────┐
  ┌────┼────┐           │ vehicle     │        │                     │
  │    │    │           └─────────────┘   HourlyPricing    WeekendPricing
Bike  Car  Truck
```

---

## Step 3: Complete Implementation

### Enums

```java
package com.parkinglot.enums;

public enum VehicleType {
    BIKE(1),
    CAR(2),
    TRUCK(3);
    
    private final int size;
    
    VehicleType(int size) {
        this.size = size;
    }
    
    public int getSize() {
        return size;
    }
}

public enum SpotType {
    BIKE(1),
    CAR(2),
    TRUCK(3);
    
    private final int size;
    
    SpotType(int size) {
        this.size = size;
    }
    
    public int getSize() {
        return size;
    }
    
    public boolean canFit(VehicleType vehicleType) {
        return this.size >= vehicleType.getSize();
    }
}

public enum SpotStatus {
    AVAILABLE,
    OCCUPIED,
    MAINTENANCE
}

public enum PaymentStatus {
    PENDING,
    COMPLETED,
    FAILED
}
```

---

### Exceptions

```java
package com.parkinglot.exceptions;

public class ParkingLotException extends RuntimeException {
    public ParkingLotException(String message) {
        super(message);
    }
}

public class ParkingLotFullException extends ParkingLotException {
    private final VehicleType vehicleType;
    
    public ParkingLotFullException(VehicleType type) {
        super("No parking spots available for " + type);
        this.vehicleType = type;
    }
    
    public VehicleType getVehicleType() {
        return vehicleType;
    }
}

public class InvalidTicketException extends ParkingLotException {
    public InvalidTicketException(String ticketId) {
        super("Invalid or expired ticket: " + ticketId);
    }
}

public class VehicleAlreadyParkedException extends ParkingLotException {
    public VehicleAlreadyParkedException(String licensePlate) {
        super("Vehicle already parked: " + licensePlate);
    }
}

public class SpotNotAvailableException extends ParkingLotException {
    public SpotNotAvailableException(String spotId) {
        super("Spot not available: " + spotId);
    }
}
```

---

### Vehicle Classes (Factory Pattern)

```java
package com.parkinglot.models;

public abstract class Vehicle {
    protected final String licensePlate;
    protected final VehicleType type;
    
    protected Vehicle(String licensePlate, VehicleType type) {
        if (licensePlate == null || licensePlate.isBlank()) {
            throw new IllegalArgumentException("License plate cannot be empty");
        }
        this.licensePlate = licensePlate.toUpperCase();
        this.type = type;
    }
    
    public String getLicensePlate() {
        return licensePlate;
    }
    
    public VehicleType getType() {
        return type;
    }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Vehicle)) return false;
        Vehicle vehicle = (Vehicle) o;
        return licensePlate.equals(vehicle.licensePlate);
    }
    
    @Override
    public int hashCode() {
        return licensePlate.hashCode();
    }
    
    @Override
    public String toString() {
        return type + "[" + licensePlate + "]";
    }
}

public class Bike extends Vehicle {
    public Bike(String licensePlate) {
        super(licensePlate, VehicleType.BIKE);
    }
}

public class Car extends Vehicle {
    public Car(String licensePlate) {
        super(licensePlate, VehicleType.CAR);
    }
}

public class Truck extends Vehicle {
    public Truck(String licensePlate) {
        super(licensePlate, VehicleType.TRUCK);
    }
}

// Factory
public class VehicleFactory {
    
    public static Vehicle createVehicle(VehicleType type, String licensePlate) {
        switch (type) {
            case BIKE:
                return new Bike(licensePlate);
            case CAR:
                return new Car(licensePlate);
            case TRUCK:
                return new Truck(licensePlate);
            default:
                throw new IllegalArgumentException("Unknown vehicle type: " + type);
        }
    }
}
```

---

### ParkingSpot

```java
package com.parkinglot.models;

import java.util.concurrent.locks.ReentrantLock;

public class ParkingSpot {
    private final String spotId;
    private final SpotType spotType;
    private final int floorNumber;
    private final ReentrantLock lock;
    
    private SpotStatus status;
    private Vehicle vehicle;
    
    public ParkingSpot(String spotId, SpotType spotType, int floorNumber) {
        this.spotId = spotId;
        this.spotType = spotType;
        this.floorNumber = floorNumber;
        this.status = SpotStatus.AVAILABLE;
        this.lock = new ReentrantLock();
    }
    
    /**
     * Attempts to park a vehicle in this spot.
     * Thread-safe operation using lock.
     */
    public boolean park(Vehicle vehicle) {
        lock.lock();
        try {
            if (!isAvailable()) {
                return false;
            }
            if (!canFit(vehicle)) {
                return false;
            }
            this.vehicle = vehicle;
            this.status = SpotStatus.OCCUPIED;
            return true;
        } finally {
            lock.unlock();
        }
    }
    
    /**
     * Removes vehicle from spot and makes it available.
     * Returns the parked vehicle.
     */
    public Vehicle unpark() {
        lock.lock();
        try {
            Vehicle parkedVehicle = this.vehicle;
            this.vehicle = null;
            this.status = SpotStatus.AVAILABLE;
            return parkedVehicle;
        } finally {
            lock.unlock();
        }
    }
    
    public boolean isAvailable() {
        return status == SpotStatus.AVAILABLE;
    }
    
    public boolean canFit(Vehicle vehicle) {
        return spotType.canFit(vehicle.getType());
    }
    
    // Getters
    public String getSpotId() { return spotId; }
    public SpotType getSpotType() { return spotType; }
    public int getFloorNumber() { return floorNumber; }
    public SpotStatus getStatus() { return status; }
    public Vehicle getVehicle() { return vehicle; }
    
    public void setStatus(SpotStatus status) {
        lock.lock();
        try {
            this.status = status;
        } finally {
            lock.unlock();
        }
    }
    
    @Override
    public String toString() {
        return String.format("Spot[%s, %s, Floor-%d, %s]", 
            spotId, spotType, floorNumber, status);
    }
}
```

---

### Ticket

```java
package com.parkinglot.models;

import java.time.LocalDateTime;
import java.time.temporal.ChronoUnit;
import java.util.UUID;

public class Ticket {
    private final String ticketId;
    private final Vehicle vehicle;
    private final ParkingSpot spot;
    private final LocalDateTime entryTime;
    private LocalDateTime exitTime;
    
    public Ticket(Vehicle vehicle, ParkingSpot spot) {
        this.ticketId = generateTicketId();
        this.vehicle = vehicle;
        this.spot = spot;
        this.entryTime = LocalDateTime.now();
    }
    
    private String generateTicketId() {
        return "TKT-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
    }
    
    public void markExit() {
        this.exitTime = LocalDateTime.now();
    }
    
    public long getParkingDurationHours() {
        LocalDateTime end = (exitTime != null) ? exitTime : LocalDateTime.now();
        long hours = ChronoUnit.HOURS.between(entryTime, end);
        return Math.max(1, hours);  // Minimum 1 hour
    }
    
    // Getters
    public String getTicketId() { return ticketId; }
    public Vehicle getVehicle() { return vehicle; }
    public ParkingSpot getSpot() { return spot; }
    public LocalDateTime getEntryTime() { return entryTime; }
    public LocalDateTime getExitTime() { return exitTime; }
    
    @Override
    public String toString() {
        return String.format("Ticket[%s, %s, %s, Entry: %s]",
            ticketId, vehicle, spot.getSpotId(), entryTime);
    }
}
```

---

### Payment

```java
package com.parkinglot.models;

import java.time.LocalDateTime;
import java.util.UUID;

public class Payment {
    private final String paymentId;
    private final Ticket ticket;
    private final double amount;
    private final LocalDateTime paymentTime;
    private PaymentStatus status;
    
    public Payment(Ticket ticket, double amount) {
        this.paymentId = "PAY-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
        this.ticket = ticket;
        this.amount = amount;
        this.paymentTime = LocalDateTime.now();
        this.status = PaymentStatus.PENDING;
    }
    
    public void complete() {
        this.status = PaymentStatus.COMPLETED;
    }
    
    public void fail() {
        this.status = PaymentStatus.FAILED;
    }
    
    // Getters
    public String getPaymentId() { return paymentId; }
    public Ticket getTicket() { return ticket; }
    public double getAmount() { return amount; }
    public LocalDateTime getPaymentTime() { return paymentTime; }
    public PaymentStatus getStatus() { return status; }
    
    @Override
    public String toString() {
        return String.format("Payment[%s, Amount: $%.2f, Status: %s]",
            paymentId, amount, status);
    }
}
```

---

### ParkingFloor

```java
package com.parkinglot.models;

import java.util.*;
import java.util.concurrent.ConcurrentLinkedQueue;
import java.util.concurrent.atomic.AtomicInteger;

public class ParkingFloor {
    private final int floorNumber;
    private final List<ParkingSpot> allSpots;
    private final Map<SpotType, Queue<ParkingSpot>> availableSpots;
    private final Map<SpotType, AtomicInteger> availableCount;
    
    public ParkingFloor(int floorNumber) {
        this.floorNumber = floorNumber;
        this.allSpots = Collections.synchronizedList(new ArrayList<>());
        this.availableSpots = new EnumMap<>(SpotType.class);
        this.availableCount = new EnumMap<>(SpotType.class);
        
        for (SpotType type : SpotType.values()) {
            availableSpots.put(type, new ConcurrentLinkedQueue<>());
            availableCount.put(type, new AtomicInteger(0));
        }
    }
    
    public void addSpot(ParkingSpot spot) {
        allSpots.add(spot);
        if (spot.isAvailable()) {
            availableSpots.get(spot.getSpotType()).offer(spot);
            availableCount.get(spot.getSpotType()).incrementAndGet();
        }
    }
    
    /**
     * Gets an available spot for the given vehicle type.
     * Uses exact match first, then tries larger spots.
     */
    public ParkingSpot getAvailableSpot(VehicleType vehicleType) {
        SpotType requiredType = SpotType.valueOf(vehicleType.name());
        
        // Try exact match first
        ParkingSpot spot = availableSpots.get(requiredType).poll();
        if (spot != null) {
            availableCount.get(requiredType).decrementAndGet();
            return spot;
        }
        
        // Try larger spots (car can park in truck spot, etc.)
        for (SpotType type : SpotType.values()) {
            if (type.canFit(vehicleType) && type != requiredType) {
                spot = availableSpots.get(type).poll();
                if (spot != null) {
                    availableCount.get(type).decrementAndGet();
                    return spot;
                }
            }
        }
        
        return null;
    }
    
    public void releaseSpot(ParkingSpot spot) {
        SpotType type = spot.getSpotType();
        availableSpots.get(type).offer(spot);
        availableCount.get(type).incrementAndGet();
    }
    
    public int getAvailableCount(VehicleType vehicleType) {
        int count = 0;
        for (SpotType type : SpotType.values()) {
            if (type.canFit(vehicleType)) {
                count += availableCount.get(type).get();
            }
        }
        return count;
    }
    
    // Getters
    public int getFloorNumber() { return floorNumber; }
    public List<ParkingSpot> getAllSpots() { return new ArrayList<>(allSpots); }
}
```

---

### Pricing Strategy (Strategy Pattern)

```java
package com.parkinglot.strategies;

import com.parkinglot.models.Ticket;

public interface PricingStrategy {
    double calculateFee(Ticket ticket);
}

public class HourlyPricingStrategy implements PricingStrategy {
    private final Map<VehicleType, Double> hourlyRates;
    
    public HourlyPricingStrategy() {
        hourlyRates = new EnumMap<>(VehicleType.class);
        hourlyRates.put(VehicleType.BIKE, 1.0);
        hourlyRates.put(VehicleType.CAR, 2.0);
        hourlyRates.put(VehicleType.TRUCK, 3.0);
    }
    
    public HourlyPricingStrategy(Map<VehicleType, Double> rates) {
        this.hourlyRates = new EnumMap<>(rates);
    }
    
    @Override
    public double calculateFee(Ticket ticket) {
        long hours = ticket.getParkingDurationHours();
        double rate = hourlyRates.get(ticket.getVehicle().getType());
        return hours * rate;
    }
}

public class FlatRatePricingStrategy implements PricingStrategy {
    private final Map<VehicleType, Double> flatRates;
    
    public FlatRatePricingStrategy() {
        flatRates = new EnumMap<>(VehicleType.class);
        flatRates.put(VehicleType.BIKE, 5.0);
        flatRates.put(VehicleType.CAR, 10.0);
        flatRates.put(VehicleType.TRUCK, 15.0);
    }
    
    @Override
    public double calculateFee(Ticket ticket) {
        return flatRates.get(ticket.getVehicle().getType());
    }
}

public class WeekendPricingStrategy implements PricingStrategy {
    private final PricingStrategy baseStrategy;
    private final double weekendMultiplier;
    
    public WeekendPricingStrategy(PricingStrategy baseStrategy, double multiplier) {
        this.baseStrategy = baseStrategy;
        this.weekendMultiplier = multiplier;
    }
    
    @Override
    public double calculateFee(Ticket ticket) {
        double baseFee = baseStrategy.calculateFee(ticket);
        
        DayOfWeek day = ticket.getEntryTime().getDayOfWeek();
        if (day == DayOfWeek.SATURDAY || day == DayOfWeek.SUNDAY) {
            return baseFee * weekendMultiplier;
        }
        
        return baseFee;
    }
}
```

---

### ParkingLot (Singleton + Facade)

```java
package com.parkinglot;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

public class ParkingLot {
    private static volatile ParkingLot instance;
    
    private final String name;
    private final List<ParkingFloor> floors;
    private final Map<String, Ticket> activeTickets;          // ticketId -> Ticket
    private final Map<String, Ticket> vehicleToTicket;        // licensePlate -> Ticket
    private PricingStrategy pricingStrategy;
    
    private ParkingLot(String name) {
        this.name = name;
        this.floors = Collections.synchronizedList(new ArrayList<>());
        this.activeTickets = new ConcurrentHashMap<>();
        this.vehicleToTicket = new ConcurrentHashMap<>();
        this.pricingStrategy = new HourlyPricingStrategy();
    }
    
    public static ParkingLot getInstance() {
        if (instance == null) {
            synchronized (ParkingLot.class) {
                if (instance == null) {
                    instance = new ParkingLot("Main Parking Lot");
                }
            }
        }
        return instance;
    }
    
    // For testing - reset singleton
    public static void resetInstance() {
        synchronized (ParkingLot.class) {
            instance = null;
        }
    }
    
    // ============ CORE OPERATIONS ============
    
    /**
     * Parks a vehicle and returns a ticket.
     * 
     * @param vehicle the vehicle to park
     * @return Ticket for the parked vehicle
     * @throws VehicleAlreadyParkedException if vehicle is already parked
     * @throws ParkingLotFullException if no spots available
     */
    public Ticket parkVehicle(Vehicle vehicle) {
        // Check if already parked
        if (vehicleToTicket.containsKey(vehicle.getLicensePlate())) {
            throw new VehicleAlreadyParkedException(vehicle.getLicensePlate());
        }
        
        // Find available spot
        ParkingSpot spot = findAvailableSpot(vehicle.getType());
        if (spot == null) {
            throw new ParkingLotFullException(vehicle.getType());
        }
        
        // Park vehicle
        if (!spot.park(vehicle)) {
            // Spot was taken between find and park (race condition)
            throw new SpotNotAvailableException(spot.getSpotId());
        }
        
        // Create ticket
        Ticket ticket = new Ticket(vehicle, spot);
        activeTickets.put(ticket.getTicketId(), ticket);
        vehicleToTicket.put(vehicle.getLicensePlate(), ticket);
        
        System.out.println("Vehicle parked: " + vehicle + " at " + spot.getSpotId());
        return ticket;
    }
    
    /**
     * Unparks a vehicle and returns payment details.
     * 
     * @param ticketId the parking ticket ID
     * @return Payment with calculated fee
     * @throws InvalidTicketException if ticket is invalid or already used
     */
    public Payment unparkVehicle(String ticketId) {
        Ticket ticket = activeTickets.remove(ticketId);
        if (ticket == null) {
            throw new InvalidTicketException(ticketId);
        }
        
        // Mark exit time
        ticket.markExit();
        
        // Unpark from spot
        ParkingSpot spot = ticket.getSpot();
        spot.unpark();
        
        // Return spot to available pool
        ParkingFloor floor = floors.get(spot.getFloorNumber());
        floor.releaseSpot(spot);
        
        // Remove vehicle tracking
        vehicleToTicket.remove(ticket.getVehicle().getLicensePlate());
        
        // Calculate fee
        double fee = pricingStrategy.calculateFee(ticket);
        Payment payment = new Payment(ticket, fee);
        payment.complete();
        
        System.out.println("Vehicle unparked: " + ticket.getVehicle() + 
                          ", Fee: $" + String.format("%.2f", fee));
        return payment;
    }
    
    /**
     * Finds an available spot for the given vehicle type.
     */
    private ParkingSpot findAvailableSpot(VehicleType type) {
        for (ParkingFloor floor : floors) {
            ParkingSpot spot = floor.getAvailableSpot(type);
            if (spot != null) {
                return spot;
            }
        }
        return null;
    }
    
    // ============ QUERY OPERATIONS ============
    
    public int getAvailableSpotsCount(VehicleType type) {
        int count = 0;
        for (ParkingFloor floor : floors) {
            count += floor.getAvailableCount(type);
        }
        return count;
    }
    
    public int getTotalSpotsCount() {
        return floors.stream()
            .mapToInt(f -> f.getAllSpots().size())
            .sum();
    }
    
    public boolean isVehicleParked(String licensePlate) {
        return vehicleToTicket.containsKey(licensePlate.toUpperCase());
    }
    
    public Optional<Ticket> getTicketByLicensePlate(String licensePlate) {
        return Optional.ofNullable(vehicleToTicket.get(licensePlate.toUpperCase()));
    }
    
    // ============ CONFIGURATION ============
    
    public void addFloor(ParkingFloor floor) {
        floors.add(floor);
        System.out.println("Added floor " + floor.getFloorNumber() + 
                          " with " + floor.getAllSpots().size() + " spots");
    }
    
    public void setPricingStrategy(PricingStrategy strategy) {
        this.pricingStrategy = strategy;
    }
    
    // Getters
    public String getName() { return name; }
    public List<ParkingFloor> getFloors() { return new ArrayList<>(floors); }
    public int getActiveTicketsCount() { return activeTickets.size(); }
}
```

---

### Parking Lot Builder

```java
package com.parkinglot;

public class ParkingLotBuilder {
    
    /**
     * Creates a parking lot with specified configuration.
     * 
     * @param numFloors number of floors
     * @param bikeSpotsPerFloor bike spots per floor
     * @param carSpotsPerFloor car spots per floor
     * @param truckSpotsPerFloor truck spots per floor
     */
    public static void initializeParkingLot(int numFloors, 
                                            int bikeSpotsPerFloor,
                                            int carSpotsPerFloor,
                                            int truckSpotsPerFloor) {
        ParkingLot.resetInstance();
        ParkingLot lot = ParkingLot.getInstance();
        
        for (int f = 0; f < numFloors; f++) {
            ParkingFloor floor = new ParkingFloor(f);
            int spotNumber = 1;
            
            // Add bike spots
            for (int i = 0; i < bikeSpotsPerFloor; i++) {
                String spotId = String.format("F%d-B%03d", f, spotNumber++);
                floor.addSpot(new ParkingSpot(spotId, SpotType.BIKE, f));
            }
            
            // Add car spots
            for (int i = 0; i < carSpotsPerFloor; i++) {
                String spotId = String.format("F%d-C%03d", f, spotNumber++);
                floor.addSpot(new ParkingSpot(spotId, SpotType.CAR, f));
            }
            
            // Add truck spots
            for (int i = 0; i < truckSpotsPerFloor; i++) {
                String spotId = String.format("F%d-T%03d", f, spotNumber++);
                floor.addSpot(new ParkingSpot(spotId, SpotType.TRUCK, f));
            }
            
            lot.addFloor(floor);
        }
        
        System.out.println("Parking lot initialized with " + numFloors + " floors");
    }
}
```

---

### Display Board (Observer Pattern)

```java
package com.parkinglot.display;

import java.util.*;

// Observer interface
public interface ParkingObserver {
    void onParkingUpdate(ParkingUpdate update);
}

// Update event
public class ParkingUpdate {
    private final Map<VehicleType, Integer> availableSpots;
    private final int totalParked;
    
    public ParkingUpdate(Map<VehicleType, Integer> availableSpots, int totalParked) {
        this.availableSpots = availableSpots;
        this.totalParked = totalParked;
    }
    
    public int getAvailable(VehicleType type) {
        return availableSpots.getOrDefault(type, 0);
    }
    
    public int getTotalParked() { return totalParked; }
}

// Display board implementation
public class DisplayBoard implements ParkingObserver {
    private final String location;
    
    public DisplayBoard(String location) {
        this.location = location;
    }
    
    @Override
    public void onParkingUpdate(ParkingUpdate update) {
        System.out.println("\n╔════════════════════════════════════╗");
        System.out.println("║     PARKING AVAILABILITY           ║");
        System.out.println("║     Location: " + padRight(location, 20) + " ║");
        System.out.println("╠════════════════════════════════════╣");
        System.out.printf("║  🏍️  Bike Spots:    %4d available  ║%n", 
            update.getAvailable(VehicleType.BIKE));
        System.out.printf("║  🚗  Car Spots:     %4d available  ║%n", 
            update.getAvailable(VehicleType.CAR));
        System.out.printf("║  🚛  Truck Spots:   %4d available  ║%n", 
            update.getAvailable(VehicleType.TRUCK));
        System.out.println("╠════════════════════════════════════╣");
        System.out.printf("║  Total Parked: %4d                 ║%n", 
            update.getTotalParked());
        System.out.println("╚════════════════════════════════════╝\n");
    }
    
    private String padRight(String s, int n) {
        return String.format("%-" + n + "s", s);
    }
}
```

---

### Main Application - Demo

```java
package com.parkinglot;

public class ParkingLotDemo {
    
    public static void main(String[] args) {
        System.out.println("=== PARKING LOT SYSTEM DEMO ===\n");
        
        // Initialize parking lot: 3 floors, 10 bike, 20 car, 5 truck spots per floor
        ParkingLotBuilder.initializeParkingLot(3, 10, 20, 5);
        ParkingLot parkingLot = ParkingLot.getInstance();
        
        // Set pricing strategy
        parkingLot.setPricingStrategy(new HourlyPricingStrategy());
        
        // Display initial status
        displayStatus(parkingLot);
        
        // Create vehicles
        Vehicle bike1 = VehicleFactory.createVehicle(VehicleType.BIKE, "BIKE-001");
        Vehicle car1 = VehicleFactory.createVehicle(VehicleType.CAR, "CAR-001");
        Vehicle car2 = VehicleFactory.createVehicle(VehicleType.CAR, "CAR-002");
        Vehicle truck1 = VehicleFactory.createVehicle(VehicleType.TRUCK, "TRUCK-001");
        
        System.out.println("\n--- PARKING VEHICLES ---");
        
        // Park vehicles
        Ticket bikeTicket = parkingLot.parkVehicle(bike1);
        Ticket carTicket1 = parkingLot.parkVehicle(car1);
        Ticket carTicket2 = parkingLot.parkVehicle(car2);
        Ticket truckTicket = parkingLot.parkVehicle(truck1);
        
        // Display status after parking
        displayStatus(parkingLot);
        
        // Check if vehicle is parked
        System.out.println("Is CAR-001 parked? " + parkingLot.isVehicleParked("CAR-001"));
        System.out.println("Is CAR-999 parked? " + parkingLot.isVehicleParked("CAR-999"));
        
        System.out.println("\n--- UNPARKING VEHICLES ---");
        
        // Simulate some time passing...
        // (In real scenario, time would pass naturally)
        
        // Unpark vehicles
        Payment payment1 = parkingLot.unparkVehicle(carTicket1.getTicketId());
        System.out.println("Payment completed: " + payment1);
        
        Payment payment2 = parkingLot.unparkVehicle(bikeTicket.getTicketId());
        System.out.println("Payment completed: " + payment2);
        
        // Display final status
        displayStatus(parkingLot);
        
        // Test error handling
        System.out.println("\n--- ERROR HANDLING TESTS ---");
        
        // Try to park same vehicle again
        try {
            Vehicle car2Again = VehicleFactory.createVehicle(VehicleType.CAR, "CAR-002");
            parkingLot.parkVehicle(car2Again);
        } catch (VehicleAlreadyParkedException e) {
            System.out.println("Expected error: " + e.getMessage());
        }
        
        // Try invalid ticket
        try {
            parkingLot.unparkVehicle("INVALID-TICKET");
        } catch (InvalidTicketException e) {
            System.out.println("Expected error: " + e.getMessage());
        }
        
        System.out.println("\n=== DEMO COMPLETE ===");
    }
    
    private static void displayStatus(ParkingLot lot) {
        System.out.println("\n┌─────────────────────────────────────┐");
        System.out.println("│        PARKING LOT STATUS           │");
        System.out.println("├─────────────────────────────────────┤");
        System.out.printf("│  Bike spots available:  %3d         │%n", 
            lot.getAvailableSpotsCount(VehicleType.BIKE));
        System.out.printf("│  Car spots available:   %3d         │%n", 
            lot.getAvailableSpotsCount(VehicleType.CAR));
        System.out.printf("│  Truck spots available: %3d         │%n", 
            lot.getAvailableSpotsCount(VehicleType.TRUCK));
        System.out.printf("│  Vehicles parked:       %3d         │%n", 
            lot.getActiveTicketsCount());
        System.out.println("└─────────────────────────────────────┘");
    }
}
```

---

## Step 4: Unit Tests

```java
package com.parkinglot.tests;

import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

class ParkingLotTest {
    
    @BeforeEach
    void setUp() {
        ParkingLot.resetInstance();
        ParkingLotBuilder.initializeParkingLot(2, 5, 10, 3);
    }
    
    @Test
    void testParkVehicle() {
        ParkingLot lot = ParkingLot.getInstance();
        Vehicle car = new Car("TEST-001");
        
        Ticket ticket = lot.parkVehicle(car);
        
        assertNotNull(ticket);
        assertNotNull(ticket.getTicketId());
        assertEquals(car, ticket.getVehicle());
        assertTrue(lot.isVehicleParked("TEST-001"));
    }
    
    @Test
    void testUnparkVehicle() {
        ParkingLot lot = ParkingLot.getInstance();
        Vehicle car = new Car("TEST-002");
        Ticket ticket = lot.parkVehicle(car);
        
        Payment payment = lot.unparkVehicle(ticket.getTicketId());
        
        assertNotNull(payment);
        assertTrue(payment.getAmount() > 0);
        assertEquals(PaymentStatus.COMPLETED, payment.getStatus());
        assertFalse(lot.isVehicleParked("TEST-002"));
    }
    
    @Test
    void testVehicleAlreadyParked() {
        ParkingLot lot = ParkingLot.getInstance();
        Vehicle car = new Car("TEST-003");
        lot.parkVehicle(car);
        
        assertThrows(VehicleAlreadyParkedException.class, () -> {
            Vehicle sameCar = new Car("TEST-003");
            lot.parkVehicle(sameCar);
        });
    }
    
    @Test
    void testInvalidTicket() {
        ParkingLot lot = ParkingLot.getInstance();
        
        assertThrows(InvalidTicketException.class, () -> {
            lot.unparkVehicle("INVALID-TICKET");
        });
    }
    
    @Test
    void testAvailableSpotsDecrease() {
        ParkingLot lot = ParkingLot.getInstance();
        int initialCount = lot.getAvailableSpotsCount(VehicleType.CAR);
        
        lot.parkVehicle(new Car("TEST-004"));
        
        assertEquals(initialCount - 1, lot.getAvailableSpotsCount(VehicleType.CAR));
    }
    
    @Test
    void testAvailableSpotsIncrease() {
        ParkingLot lot = ParkingLot.getInstance();
        Vehicle car = new Car("TEST-005");
        Ticket ticket = lot.parkVehicle(car);
        int countAfterPark = lot.getAvailableSpotsCount(VehicleType.CAR);
        
        lot.unparkVehicle(ticket.getTicketId());
        
        assertEquals(countAfterPark + 1, lot.getAvailableSpotsCount(VehicleType.CAR));
    }
    
    @Test
    void testPricingStrategy() {
        ParkingLot lot = ParkingLot.getInstance();
        lot.setPricingStrategy(new FlatRatePricingStrategy());
        
        Vehicle bike = new Bike("TEST-006");
        Ticket ticket = lot.parkVehicle(bike);
        Payment payment = lot.unparkVehicle(ticket.getTicketId());
        
        assertEquals(5.0, payment.getAmount()); // Flat rate for bike
    }
}
```

---

## Design Patterns Used

| Pattern | Where Used |
|---------|------------|
| **Singleton** | ParkingLot - single instance |
| **Factory** | VehicleFactory - create vehicles |
| **Strategy** | PricingStrategy - different pricing |
| **Observer** | DisplayBoard - availability updates |
| **Builder** | ParkingLotBuilder - configure parking lot |
| **Facade** | ParkingLot - simplified interface |

---

## Key Takeaways

1. **Requirements First** - Always clarify before coding
2. **Identify Entities** - Extract nouns from requirements
3. **Design Relationships** - Use UML to visualize
4. **Apply Patterns** - Use appropriate design patterns
5. **Handle Concurrency** - Thread-safe operations
6. **Write Tests** - Ensure correctness
7. **Handle Errors** - Custom exceptions for each case

---

**Congratulations! You have completed the LLD course!**
