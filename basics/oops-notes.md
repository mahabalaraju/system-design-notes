## 1. What is OOPs?

Object Oriented Programming System (OOPs) is a programming concept that works on the principles of:

- **Abstraction**
- **Encapsulation**
- **Inheritance**
- **Polymorphism**

The basic concept is to create objects, reuse them throughout the program, and manipulate these objects to get results.

---

## 2. Class

A class is a logical entity — a group of similar entities. It is NOT a physical entity.

Think of it as a **blueprint** or **template**.

```java
// Class = Blueprint
public class Car {
    // Properties (data)
    String brand;
    double speed;
    double mileage;
    double price;

    // Methods (behaviour)
    void drive() {
        System.out.println(brand + " is driving!");
    }

    void brake() {
        System.out.println(brand + " is braking!");
    }

    void reverse() {
        System.out.println(brand + " is reversing!");
    }
}
```

**Real life example:**
- Class = Expensive Cars
- Properties = speed, mileage, price
- Objects = BMW, Mercedes, Audi

---

## 3. Object

An object is an **instance of a class**. There can be multiple instances of a class.

Class is the blueprint. Object is the actual thing built from that blueprint.

```java
public class Main {
    public static void main(String[] args) {
        // Creating objects (instances) of Car class
        Car bmw = new Car();
        bmw.brand = "BMW";
        bmw.speed = 250;
        bmw.price = 5000000;
        bmw.drive(); // BMW is driving!

        Car mercedes = new Car();
        mercedes.brand = "Mercedes";
        mercedes.speed = 280;
        mercedes.price = 8000000;
        mercedes.drive(); // Mercedes is driving!
    }
}
```

**Memory picture:**
```
Car class (blueprint)
    ↓         ↓
  bmw       mercedes
(object)    (object)
```

---

## 4. Abstraction

Shows only **essential attributes** and hides unnecessary details.

Selection of data from a larger pool to show only relevant details to the user.

**Real life examples:**
- Driving a car — you use steering, accelerator, brake. You don't see the engine internals.
- ATM machine — you press buttons and get cash. You don't see backend banking logic.
- Webpage — you see UI. You don't see server code.

**Benefits:**
- Security — user can't see internal code
- Reduces complexity
- Modularity and maintainability
- Groups related classes as single entity

**In Java — achieved using Abstract classes and Interfaces**

```java
// Abstract class — partial abstraction
abstract class Vehicle {
    String brand;

    // Abstract method — no implementation, child MUST implement
    abstract void start();

    // Concrete method — has implementation
    void stop() {
        System.out.println(brand + " stopped!");
    }
}

public class Car extends Vehicle {
    @Override
    public void start() {
        System.out.println(brand + " car started with key!");
    }
}

public class Bike extends Vehicle {
    @Override
    public void start() {
        System.out.println(brand + " bike started with kick!");
    }
}
```

```java
// Interface — 100% abstraction
public interface Payable {
    void pay(double amount); // no implementation
    void refund(double amount); // no implementation
}

public class CreditCard implements Payable {
    @Override
    public void pay(double amount) {
        System.out.println("Paid " + amount + " via Credit Card");
    }

    @Override
    public void refund(double amount) {
        System.out.println("Refunded " + amount + " to Credit Card");
    }
}
```

**Abstract class vs Interface:**

| Feature | Abstract Class | Interface |
|---|---|---|
| Use when | Know partial implementation | Contract / capability |
| Variables | Any type | Only public static final |
| Variable Initialization | Not mandatory during declaration | Must be initialized during declaration |
| Inheritance | Single | Multiple |
| Instance & Static blocks | Can declare | Cannot declare |
| Constructors | Can declare | Cannot declare |
| Methods | Abstract + Concrete | Only abstract (Java 7), default allowed (Java 8+) |

---

## 5. Encapsulation

**Wrapping data (variables) and methods together** in a single unit (class) and restricting direct access.

Think of it as a **capsule** — medicine inside, protected from outside.

**Real life example:**
- Your bank account — you can't directly change your balance. You go through deposit/withdraw methods.
- A capsule tablet — medicine is inside, protected from outside.

**How to achieve in Java:**
- Make variables **private**
- Provide **public getters and setters**

```java
public class BankAccount {
    // Private — hidden from outside
    private String accountNumber;
    private double balance;
    private String owner;

    // Constructor
    public BankAccount(String accountNumber, String owner) {
        this.accountNumber = accountNumber;
        this.owner = owner;
        this.balance = 0;
    }

    // Public getter — read only
    public double getBalance() {
        return balance;
    }

    public String getOwner() {
        return owner;
    }

    // Controlled access — with validation
    public void deposit(double amount) {
        if(amount > 0) {
            balance += amount;
            System.out.println("Deposited: " + amount);
        } else {
            System.out.println("Invalid amount!");
        }
    }

    public void withdraw(double amount) {
        if(amount > 0 && amount <= balance) {
            balance -= amount;
            System.out.println("Withdrawn: " + amount);
        } else {
            System.out.println("Insufficient balance!");
        }
    }
}

// Usage
BankAccount account = new BankAccount("ACC001", "Mahabala");
account.deposit(5000);
account.withdraw(2000);
System.out.println(account.getBalance()); // 3000
// account.balance = 1000000; ❌ Not allowed — private!
```

**Benefits:**
- Data security and hiding
- Control over data — validation in setters
- Flexibility — change internal implementation without affecting outside code
- Easy to test

---

## 6. Inheritance

Its a mechanism in which one class acquires the properties and behaviours of parent class. the keyword extends is used by subclasses to inherit the features of superclass. 

A child class **acquires properties and methods** of parent class.

**"IS-A" relationship.**

**Real life example:**
- Dog IS-A Animal
- Car IS-A Vehicle
- SavingsAccount IS-A BankAccount

```java
// Parent class
public class Animal {
    String name;
    int age;

    public void eat() {
        System.out.println(name + " is eating");
    }

    public void sleep() {
        System.out.println(name + " is sleeping");
    }
}

// Child class inherits from Animal
public class Dog extends Animal {
    String breed;

    public void bark() {
        System.out.println(name + " is barking! Woof!");
    }
}

public class Cat extends Animal {
    public void meow() {
        System.out.println(name + " is meowing! Meow!");
    }
}

// Usage
Dog dog = new Dog();
dog.name = "Tommy";
dog.eat();   // inherited from Animal
dog.bark();  // Dog's own method
```

**Types of Inheritance in Java:**

```
1. Single         A → B
2. Multilevel     A → B → C
3. Hierarchical   A → B, A → C
4. Multiple       ❌ Not supported with classes  A + B → C
                  ✅ Supported with interfaces
```

```java
// Multilevel inheritance
class Animal {
    void breathe() { System.out.println("Breathing"); }
}

class Mammal extends Animal {
    void feedMilk() { System.out.println("Feeding milk"); }
}

class Dog extends Mammal {
    void bark() { System.out.println("Barking"); }
}

// Dog has: breathe() + feedMilk() + bark()
```

**Benefits:**
- Code reusability — write once, use in child classes
- Method overriding — customize parent behaviour
- Polymorphism support

---

## 7. Polymorphism

**One name, many forms.**

Same method name — different behaviour based on context.

**Real life example:**
- Person IS-A Employee at office, IS-A Father at home, IS-A Customer at shop — same person, different roles!

### 7a. Compile Time Polymorphism — Method Overloading

Same method name, **different parameters** in same class.
Decided at **compile time**.

```java
public class Calculator {
    // Same name — different parameters
    public int add(int a, int b) {
        return a + b;
    }

    public double add(double a, double b) {
        return a + b;
    }

    public int add(int a, int b, int c) {
        return a + b + c;
    }

    public String add(String a, String b) {
        return a + b;
    }
}

Calculator calc = new Calculator();
calc.add(2, 3);           // 5
calc.add(2.5, 3.5);       // 6.0
calc.add(1, 2, 3);        // 6
calc.add("Hello", "World"); // HelloWorld
```

### 7b. Runtime Polymorphism — Method Overriding

Child class **redefines** parent class method.
Decided at **runtime**.

```java
public class Shape {
    public double area() {
        return 0;
    }
}

public class Circle extends Shape {
    double radius;

    Circle(double radius) {
        this.radius = radius;
    }

    @Override
    public double area() {
        return Math.PI * radius * radius;
    }
}

public class Rectangle extends Shape {
    double length, width;

    Rectangle(double length, double width) {
        this.length = length;
        this.width = width;
    }

    @Override
    public double area() {
        return length * width;
    }
}

// Runtime polymorphism
Shape shape1 = new Circle(5);
Shape shape2 = new Rectangle(4, 6);

System.out.println(shape1.area()); // 78.53 — Circle's area
System.out.println(shape2.area()); // 24.0 — Rectangle's area
```

---

## 8. OOPs in Spring Boot — Real World Connection

| OOPs Concept | Spring Boot Usage |
|---|---|
| **Abstraction** | Interfaces for Service layer — `UserService` interface |
| **Encapsulation** | Private fields in Entity classes with getters/setters |
| **Inheritance** | `JpaRepository` extends `CrudRepository` |
| **Polymorphism** | Multiple implementations of same interface — `@Qualifier` |

```java
// Abstraction + Polymorphism in Spring Boot
public interface PaymentService {
    void processPayment(double amount);
}

@Service("creditCard")
public class CreditCardService implements PaymentService {
    @Override
    public void processPayment(double amount) {
        System.out.println("Processing credit card payment: " + amount);
    }
}

@Service("upi")
public class UPIService implements PaymentService {
    @Override
    public void processPayment(double amount) {
        System.out.println("Processing UPI payment: " + amount);
    }
}

// Controller uses abstraction — doesn't know which implementation
@RestController
public class PaymentController {
    @Autowired
    @Qualifier("upi")
    private PaymentService paymentService;

    @PostMapping("/pay")
    public void pay(@RequestParam double amount) {
        paymentService.processPayment(amount);
    }
}
```
**Method overloading vs method overriding:**

| Feature | Method Overloading | Method Overriding |
|---|---|---|
| **Method names** | Must be same | Must be same |
| **Argument types / parameters** | Must be different (number, type, or order) | Must be same (same number, type, and order) |
| **Class relationship** | Usually happens in the same class (can also occur in inheritance) | Requires inheritance (parent-child relationship) |
| **Return types** | no restriction | Must be same or covariant |
| **private / final / static methods** | Can be overloaded | `private` and `final` cannot be overridden; `static` methods are hidden, not overridden |
| **Checked exceptions** | No restriction |cannot throw broader checked exception than parent |
| **Method resolution** | Takes place at compile time | Takes place at runtime |
| **Polymorphism type** | Compile-time polymorphism | Runtime polymorphism |
| **Constructors** | Can be overloaded | Cannot be overridden |
| **Use when** | Need multiple versions of a method for different inputs | Need specific implementation of a parent class method in child class |

---
---

## 9. Key Interview Questions

**Q: What are the 4 pillars of OOPs?**
A: Abstraction, Encapsulation, Inheritance, Polymorphism.

**Q: Difference between Abstraction and Encapsulation?**
A: Abstraction = hiding complexity (WHAT). Encapsulation = hiding data (HOW). Abstraction is design level, Encapsulation is implementation level.

**Q: Difference between Abstract class and Interface?**
A: Abstract class — partial abstraction, single inheritance, can have constructors. Interface — 100% abstraction, multiple inheritance, no constructors.

**Q: What is method overloading vs overriding?**
A: Overloading = same method name, different parameters, same class, compile time. Overriding = child redefines parent method, runtime.

**Q: Can we override static methods?**
A: No. Static methods belong to class not object. They can be hidden but not overridden.

**Q: What is the difference between IS-A and HAS-A?**
A: IS-A = Inheritance (Dog IS-A Animal). HAS-A = Composition (Car HAS-A Engine).

**Q: Why is multiple inheritance not supported in Java with classes?**
A: Diamond problem — ambiguity when two parent classes have same method. Solved using interfaces.

---

## 10. My Learning Progress

- [x] Class and Object
- [x] Abstraction
- [x] Encapsulation
- [x] Inheritance
- [x] Polymorphism
- [ ] SOLID Principles
- [ ] Design Patterns
- [ ] OOPs in Microservices

---

*"Strong engineers never stop learning"* — Mahabalaraju H
