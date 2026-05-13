# SOLID Principles & ACID Properties — Complete Notes

> Advanced reference covering SOLID OOP design principles with Java examples, and ACID database transaction properties with deep internals.

---

## Table of Contents

- [SOLID Principles](#solid-principles)
  - [S — Single Responsibility Principle](#s--single-responsibility-principle-srp)
  - [O — Open/Closed Principle](#o--openclosed-principle-ocp)
  - [L — Liskov Substitution Principle](#l--liskov-substitution-principle-lsp)
  - [I — Interface Segregation Principle](#i--interface-segregation-principle-isp)
  - [D — Dependency Inversion Principle](#d--dependency-inversion-principle-dip)
  - [SOLID Quick Reference](#solid-quick-reference)
  - [SOLID Violations Cheat Sheet](#solid-violations-cheat-sheet)
- [ACID Properties](#acid-properties)
  - [Atomicity](#1-atomicity)
  - [Consistency](#2-consistency)
  - [Isolation](#3-isolation)
  - [Durability](#4-durability)
  - [Isolation Levels Deep Dive](#isolation-levels-deep-dive)
  - [ACID vs BASE](#acid-vs-base)
  - [ACID in Java (JDBC & Spring)](#acid-in-java-jdbc--spring)

---

## SOLID Principles

> Introduced by Robert C. Martin ("Uncle Bob"). Five principles that make software designs more understandable, flexible, and maintainable.

---

### S — Single Responsibility Principle (SRP)

> **"A class should have only one reason to change."**

A class should do one thing and do it well. If a class has multiple responsibilities, changes to one responsibility may break the other.

#### ❌ Violation

```java
public class UserService {
    public User findUser(Long id) { /* query DB */ return null; }
    public void saveUser(User user) { /* persist to DB */ }
    public String generateUserReport(User user) { /* build HTML report */ return ""; }
    public void sendWelcomeEmail(User user) { /* SMTP logic */ }
}
```

This class has **four reasons to change**: DB logic, report format, email template, business rules.

#### ✅ Correct

```java
public class UserRepository {
    public User findById(Long id) { /* DB query */ return null; }
    public void save(User user) { /* DB persist */ }
}

public class UserReportService {
    public String generateReport(User user) { /* only report logic */ return ""; }
}

public class UserEmailService {
    public void sendWelcomeEmail(User user) { /* only email logic */ }
}

public class UserService {
    private final UserRepository repo;
    private final UserEmailService emailService;

    public UserService(UserRepository repo, UserEmailService emailService) {
        this.repo = repo;
        this.emailService = emailService;
    }

    public void registerUser(User user) {
        repo.save(user);
        emailService.sendWelcomeEmail(user);
    }
}
```

#### Key Insight
SRP is not about having one method. It's about **one actor** (person/team/reason) that could request a change. Ask: *"Who would ask me to change this class?"* — if the answer is more than one type of stakeholder, split it.

---

### O — Open/Closed Principle (OCP)

> **"Software entities should be open for extension, but closed for modification."**

You should be able to add new behavior without changing existing, tested code.

#### ❌ Violation — adding a new shape breaks existing code

```java
public class AreaCalculator {
    public double calculate(Object shape) {
        if (shape instanceof Circle c) {
            return Math.PI * c.radius * c.radius;
        } else if (shape instanceof Rectangle r) {
            return r.width * r.height;
        }
        // Adding Triangle forces modifying this class ← violation
        throw new IllegalArgumentException("Unknown shape");
    }
}
```

#### ✅ Correct — extend by adding new classes, not modifying old ones

```java
public interface Shape {
    double area();
}

public class Circle implements Shape {
    private final double radius;
    public Circle(double radius) { this.radius = radius; }
    public double area() { return Math.PI * radius * radius; }
}

public class Rectangle implements Shape {
    private final double width, height;
    public Rectangle(double w, double h) { this.width = w; this.height = h; }
    public double area() { return width * height; }
}

// Adding Triangle requires ZERO changes to existing code
public class Triangle implements Shape {
    private final double base, height;
    public Triangle(double base, double height) { this.base = base; this.height = height; }
    public double area() { return 0.5 * base * height; }
}

public class AreaCalculator {
    public double calculate(Shape shape) { return shape.area(); } // never changes
}
```

#### Mechanisms to achieve OCP
- Polymorphism (interfaces / abstract classes)
- Strategy pattern
- Plugin architectures
- Java `ServiceLoader`

#### Key Insight
You can't anticipate every future requirement. OCP is about protecting stable code from volatile code — draw the line at what's most likely to change.

---

### L — Liskov Substitution Principle (LSP)

> **"Objects of a subclass should be replaceable with objects of the superclass without breaking the program."**

Formally: if `S` is a subtype of `T`, then objects of type `T` may be replaced with objects of type `S` without altering the correctness of the program.

#### ❌ Classic violation — the Square/Rectangle problem

```java
public class Rectangle {
    protected int width, height;

    public void setWidth(int w)  { this.width = w; }
    public void setHeight(int h) { this.height = h; }
    public int area() { return width * height; }
}

public class Square extends Rectangle {
    @Override
    public void setWidth(int w)  { this.width = w; this.height = w; } // breaks invariant!
    @Override
    public void setHeight(int h) { this.width = h; this.height = h; }
}

// Client code — assumes Rectangle behavior
void testArea(Rectangle r) {
    r.setWidth(5);
    r.setHeight(4);
    assert r.area() == 20; // FAILS when r is a Square → area is 16
}
```

#### ✅ Correct — separate hierarchy

```java
public interface Shape {
    int area();
}

public class Rectangle implements Shape {
    private final int width, height;
    public Rectangle(int w, int h) { this.width = w; this.height = h; }
    public int area() { return width * height; }
}

public class Square implements Shape {
    private final int side;
    public Square(int side) { this.side = side; }
    public int area() { return side * side; }
}
```

#### LSP Rules (Barbara Liskov's contracts)
| Rule | Meaning |
|---|---|
| **Preconditions** | Subclass cannot strengthen (narrow) preconditions |
| **Postconditions** | Subclass cannot weaken (broaden) postconditions |
| **Invariants** | Subclass must preserve superclass invariants |
| **Exceptions** | Subclass should not throw new checked exceptions |
| **History constraint** | Subclass should not allow state changes the superclass forbids |

#### Key Insight
LSP is violated when you write `instanceof` checks or `if (obj is SubType)` in client code — that's a sign substitution isn't truly working.

---

### I — Interface Segregation Principle (ISP)

> **"Clients should not be forced to depend on interfaces they do not use."**

Prefer many small, specific interfaces over one large, general-purpose one.

#### ❌ Violation — fat interface

```java
public interface Worker {
    void work();
    void eat();
    void sleep();
    void attendMeeting();
}

// Robot worker — forced to implement irrelevant methods
public class RobotWorker implements Worker {
    public void work()          { /* valid */ }
    public void eat()           { throw new UnsupportedOperationException(); } // violation!
    public void sleep()         { throw new UnsupportedOperationException(); } // violation!
    public void attendMeeting() { /* maybe valid */ }
}
```

#### ✅ Correct — segregated interfaces

```java
public interface Workable    { void work(); }
public interface Eatable     { void eat(); }
public interface Sleepable   { void sleep(); }
public interface Meetable    { void attendMeeting(); }

public class HumanWorker implements Workable, Eatable, Sleepable, Meetable {
    public void work()          { /* ... */ }
    public void eat()           { /* ... */ }
    public void sleep()         { /* ... */ }
    public void attendMeeting() { /* ... */ }
}

public class RobotWorker implements Workable, Meetable {
    public void work()          { /* ... */ }
    public void attendMeeting() { /* ... */ }
    // no forced irrelevant methods
}
```

#### Real-world Java examples
- `java.io.Readable` vs `java.io.Closeable` vs `java.io.Flushable` — not one giant `IO` interface
- Spring's `ApplicationContext` vs `BeanFactory` — different capability levels

#### Key Insight
ISP and SRP are related but different. SRP is about a class having one reason to change. ISP is about an interface not forcing irrelevant method implementations on clients.

---

### D — Dependency Inversion Principle (DIP)

> **"High-level modules should not depend on low-level modules. Both should depend on abstractions."**
> **"Abstractions should not depend on details. Details should depend on abstractions."**

#### ❌ Violation — high-level depends on low-level concrete class

```java
public class OrderService {
    private final MySQLOrderRepository repository; // concrete dependency ← violation

    public OrderService() {
        this.repository = new MySQLOrderRepository(); // tight coupling
    }

    public void placeOrder(Order order) {
        repository.save(order);
    }
}
```

Switching from MySQL to PostgreSQL or a mock in tests requires changing `OrderService`.

#### ✅ Correct — depend on abstraction

```java
// Abstraction (interface)
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(Long id);
}

// Low-level detail depends on abstraction
public class MySQLOrderRepository implements OrderRepository {
    public void save(Order order) { /* MySQL logic */ }
    public Optional<Order> findById(Long id) { /* MySQL query */ return Optional.empty(); }
}

public class InMemoryOrderRepository implements OrderRepository {
    private final Map<Long, Order> store = new HashMap<>();
    public void save(Order order) { store.put(order.getId(), order); }
    public Optional<Order> findById(Long id) { return Optional.ofNullable(store.get(id)); }
}

// High-level module depends on abstraction — not the detail
public class OrderService {
    private final OrderRepository repository; // interface, not concrete class

    // Dependency injected from outside
    public OrderService(OrderRepository repository) {
        this.repository = repository;
    }

    public void placeOrder(Order order) {
        repository.save(order);
    }
}

// Wiring (in main / DI container)
OrderService service = new OrderService(new MySQLOrderRepository());
// or for tests:
OrderService testService = new OrderService(new InMemoryOrderRepository());
```

#### DIP vs Dependency Injection
- **DIP** is the *principle* — depend on abstractions
- **Dependency Injection** is a *pattern/technique* that implements DIP
- **IoC Containers** (Spring, Guice) automate DI at scale

#### Key Insight
DIP flips the traditional top-down dependency flow. Without it, your business logic becomes tightly coupled to infrastructure — the hardest thing to swap.

---

### SOLID Quick Reference

| Principle | One-liner | Violation smell | Fix |
|---|---|---|---|
| **S**RP | One reason to change | God class, multiple stakeholders | Split into focused classes |
| **O**CP | Extend, don't modify | `if/else` or `switch` on type | Polymorphism / Strategy |
| **L**SP | Subtypes must substitute | `instanceof` checks, `UnsupportedOperationException` | Redesign hierarchy |
| **I**SP | Don't force unused methods | Fat interfaces, empty implementations | Split interfaces |
| **D**IP | Depend on abstractions | `new ConcreteClass()` in business logic | Inject via interface |

---

### SOLID Violations Cheat Sheet

```
God Class                → SRP violation
Switch on type           → OCP violation
Square extends Rectangle → LSP violation
Marker interface abuse   → ISP violation
new MySQLRepo() in service → DIP violation
```

---

## ACID Properties

> ACID defines the properties that guarantee database transactions are processed reliably, even in the face of errors, power failures, and concurrency.

---

### 1. Atomicity

> **"All or nothing."**

A transaction is treated as a single unit — either **all operations succeed** or **none of them are applied**.

#### Example

```sql
-- Transfer ₹1000 from Account A to Account B
BEGIN TRANSACTION;
    UPDATE accounts SET balance = balance - 1000 WHERE id = 'A';
    UPDATE accounts SET balance = balance + 1000 WHERE id = 'B';
COMMIT;
```

If the second `UPDATE` fails (e.g., network error, crash), the entire transaction is **rolled back** — Account A is not debited.

#### How databases implement it
- **Undo Log (Rollback Segment)** — before-image of data is stored; on failure, undo log is replayed in reverse
- **Write-Ahead Log (WAL)** — changes logged before applied; incomplete transactions are undone on recovery
- **Shadow Paging** — maintains two versions; only the committed version becomes active

#### Java (JDBC)

```java
Connection conn = dataSource.getConnection();
conn.setAutoCommit(false); // disable auto-commit → begin transaction

try {
    PreparedStatement debit  = conn.prepareStatement("UPDATE accounts SET balance = balance - ? WHERE id = ?");
    PreparedStatement credit = conn.prepareStatement("UPDATE accounts SET balance = balance + ? WHERE id = ?");

    debit.setDouble(1, 1000); debit.setString(2, "A"); debit.executeUpdate();
    credit.setDouble(1, 1000); credit.setString(2, "B"); credit.executeUpdate();

    conn.commit(); // atomically applied
} catch (SQLException e) {
    conn.rollback(); // atomically undone
    throw e;
} finally {
    conn.setAutoCommit(true);
    conn.close();
}
```

---

### 2. Consistency

> **"A transaction brings the database from one valid state to another valid state."**

All data written must satisfy defined rules: constraints, cascades, triggers, and application-level invariants.

#### Types of consistency rules

| Rule type | Example |
|---|---|
| **Entity integrity** | Primary key must be unique and not null |
| **Referential integrity** | Foreign key must reference an existing row |
| **Domain constraints** | `balance >= 0`, `age BETWEEN 0 AND 150` |
| **Business invariants** | Total debits must equal total credits |
| **Check constraints** | `CHECK (salary > 0)` |

#### Example

```sql
ALTER TABLE accounts ADD CONSTRAINT chk_balance CHECK (balance >= 0);

-- This transaction will be REJECTED by the DB — consistency preserved
BEGIN TRANSACTION;
    UPDATE accounts SET balance = balance - 9999999 WHERE id = 'A'; -- violates CHECK
COMMIT; -- ← never reached; constraint error thrown
```

#### Key Insight
Consistency is partially the **database's job** (constraints) and partially the **application's job** (business rules). The DB enforces structural consistency; your code enforces semantic consistency.

---

### 3. Isolation

> **"Concurrent transactions execute as if they were serial."**

Transactions should not interfere with each other even when running simultaneously.

#### Concurrency problems isolation must prevent

| Problem | Description | Example |
|---|---|---|
| **Dirty Read** | Read uncommitted data from another transaction | T2 reads balance updated by T1 before T1 commits; T1 rolls back |
| **Non-repeatable Read** | Same row read twice gives different values | T1 reads row; T2 updates & commits; T1 reads again → different |
| **Phantom Read** | Re-running a query returns new/missing rows | T1 queries `WHERE age > 30`; T2 inserts a row; T1 re-queries → new row appears |
| **Lost Update** | Two transactions update same row; one update is lost | T1 and T2 both read balance=100; T1 writes 150; T2 writes 120 → T1's update lost |
| **Write Skew** | Each transaction reads consistent data, but combined writes violate an invariant | Two doctors both see ≥2 on-call; both go off-call simultaneously → 0 on-call |

---

### Isolation Levels Deep Dive

SQL standard defines four levels (from weakest to strongest):

#### Level 1 — Read Uncommitted (lowest isolation)

```
Prevents: nothing meaningful
Allows: Dirty Reads, Non-repeatable Reads, Phantom Reads
Use: analytics where approximate data is acceptable, bulk reads
```

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT * FROM orders; -- may see rows from uncommitted transactions
```

#### Level 2 — Read Committed (default in PostgreSQL, Oracle, SQL Server)

```
Prevents: Dirty Reads
Allows: Non-repeatable Reads, Phantom Reads
Use: most OLTP workloads
```

Each `SELECT` sees only committed data at the moment that statement executes.

#### Level 3 — Repeatable Read (default in MySQL InnoDB)

```
Prevents: Dirty Reads, Non-repeatable Reads
Allows: Phantom Reads (in standard SQL; MySQL prevents them via gap locks)
Use: reports that re-read the same rows, financial summaries
```

A snapshot is taken at the start of the transaction; all reads see that snapshot.

#### Level 4 — Serializable (strongest isolation)

```
Prevents: ALL anomalies including Phantom Reads and Write Skew
Use: financial transactions, inventory management, anything requiring full correctness
Cost: highest — uses range locks or MVCC with conflict detection
```

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

#### Isolation Level Summary Table

| Isolation Level | Dirty Read | Non-repeatable Read | Phantom Read | Write Skew |
|---|---|---|---|---|
| Read Uncommitted | ✅ Possible | ✅ Possible | ✅ Possible | ✅ Possible |
| Read Committed | ❌ Prevented | ✅ Possible | ✅ Possible | ✅ Possible |
| Repeatable Read | ❌ Prevented | ❌ Prevented | ✅ Possible* | ✅ Possible |
| Serializable | ❌ Prevented | ❌ Prevented | ❌ Prevented | ❌ Prevented |

> *MySQL InnoDB prevents Phantom Reads even at Repeatable Read via next-key locking.

#### Implementation mechanisms

**Pessimistic Concurrency (Lock-based):**
```
Shared Lock (S-Lock)    → acquired on reads; multiple allowed
Exclusive Lock (X-Lock) → acquired on writes; blocks all others
Range Lock / Gap Lock   → prevents phantom reads by locking key ranges
```

**Optimistic Concurrency (MVCC — Multi-Version Concurrency Control):**
```
Each write creates a new version with a timestamp/transaction ID
Readers see a consistent snapshot without locking
PostgreSQL, MySQL InnoDB, Oracle use MVCC
Writers check for conflicts at commit time
```

---

### 4. Durability

> **"Once a transaction is committed, it remains committed — even in the face of crashes, power loss, or errors."**

#### How databases implement durability

**Write-Ahead Logging (WAL):**
```
1. Changes are written to the WAL (on durable storage) BEFORE the actual data pages
2. On commit → WAL entry is fsynced to disk
3. On crash recovery → WAL is replayed to restore committed state
4. Data pages may be dirty in memory; WAL guarantees no committed data is lost
```

**Checkpointing:**
```
Periodically flush dirty buffer pool pages to disk
Reduces WAL replay time on recovery
Background process in PostgreSQL, InnoDB
```

**Replication for durability:**
```
Synchronous replication → commit confirmed only after replica acknowledges write
Asynchronous replication → faster but potential data loss on primary crash
```

#### Java — ensuring durability

```java
// JDBC — fsync happens automatically on conn.commit()
conn.commit(); // durable after this line

// Spring — @Transactional ensures commit/rollback semantics
@Service
public class PaymentService {

    @Transactional // begins txn, commits on success, rolls back on RuntimeException
    public void processPayment(Payment payment) {
        accountRepo.debit(payment.getFromAccount(), payment.getAmount());
        accountRepo.credit(payment.getToAccount(), payment.getAmount());
        auditLog.record(payment);
    }

    @Transactional(rollbackFor = PaymentException.class) // roll back on checked exception too
    public void processPaymentStrict(Payment payment) throws PaymentException { /* ... */ }

    @Transactional(isolation = Isolation.SERIALIZABLE) // explicit isolation level
    public void criticalTransfer(Transfer t) { /* ... */ }

    @Transactional(propagation = Propagation.REQUIRES_NEW) // always new transaction
    public void auditEntry(AuditRecord record) { /* ... */ }
}
```

#### `@Transactional` Propagation Levels

| Propagation | Behavior |
|---|---|
| `REQUIRED` (default) | Join existing txn or create new one |
| `REQUIRES_NEW` | Always create new txn; suspend existing |
| `NESTED` | Savepoint within existing txn |
| `MANDATORY` | Must join existing txn; throw if none |
| `SUPPORTS` | Join if exists; run non-transactionally if not |
| `NOT_SUPPORTED` | Suspend existing txn; run non-transactionally |
| `NEVER` | Throw if a txn exists |

---

### ACID vs BASE

NoSQL systems often trade ACID for availability and performance, following BASE:

| Property | ACID | BASE |
|---|---|---|
| **Focus** | Strong consistency | Availability |
| **Model** | Relational (PostgreSQL, MySQL, Oracle) | Distributed (Cassandra, DynamoDB, MongoDB) |
| **Consistency** | Immediate, strict | Eventually consistent |
| **Availability** | May reject requests to maintain consistency | Always available |
| **Transactions** | Full ACID | Limited; often per-document |
| **Acronym** | Atomicity, Consistency, Isolation, Durability | **B**asically **A**vailable, **S**oft state, **E**ventual consistency |

> **CAP Theorem:** A distributed system can guarantee at most two of: Consistency, Availability, Partition Tolerance. ACID databases choose CP; BASE systems choose AP.

---

### ACID in Java (JDBC & Spring)

#### JDBC transaction template

```java
public <T> T executeInTransaction(Connection conn, Callable<T> work) throws Exception {
    conn.setAutoCommit(false);
    Savepoint savepoint = conn.setSavepoint(); // optional partial rollback point

    try {
        T result = work.call();
        conn.commit();
        return result;
    } catch (Exception e) {
        conn.rollback(savepoint); // or conn.rollback() for full rollback
        throw e;
    } finally {
        conn.setAutoCommit(true);
    }
}
```

#### Spring `@Transactional` common pitfalls

```java
@Service
public class OrderService {

    // ❌ PITFALL 1: self-invocation — @Transactional bypassed (no proxy)
    public void processOrders() {
        this.placeOrder(new Order()); // calls the real method, not the proxy
    }

    @Transactional
    public void placeOrder(Order order) { /* ... */ }


    // ❌ PITFALL 2: checked exceptions don't trigger rollback by default
    @Transactional
    public void riskyOp() throws IOException {
        repo.save(entity);
        throw new IOException("File missing"); // transaction COMMITS, not rolls back!
    }

    // ✅ FIX: declare rollbackFor
    @Transactional(rollbackFor = IOException.class)
    public void riskyOpFixed() throws IOException { /* ... */ }


    // ❌ PITFALL 3: @Transactional on private methods — ignored by Spring AOP proxy
    @Transactional
    private void internalSave() { /* not transactional! */ }
}
```

---

## Summary

### SOLID in one sentence each

- **SRP** — Each class has one job, owned by one team/stakeholder.
- **OCP** — Add new features by adding new code, not editing existing code.
- **LSP** — A subclass must honour every contract of its superclass.
- **ISP** — Don't force clients to implement methods they don't need.
- **DIP** — Business logic depends on interfaces; infrastructure depends on those same interfaces.

### ACID in one sentence each

- **Atomicity** — The whole transaction succeeds or nothing happens.
- **Consistency** — Every transaction moves the DB from one valid state to another.
- **Isolation** — Concurrent transactions don't see each other's intermediate state.
- **Durability** — Committed data survives crashes, guaranteed by WAL and replication.

---

*References: Robert C. Martin — Clean Architecture (2017) | E.F. Codd — A Relational Model of Data (1970) | Jim Gray — The Transaction Concept (1981)*
