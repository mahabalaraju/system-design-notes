# Spring Data JPA — Complete Notes

> Advanced reference covering Spring Data JPA from entity mapping to performance tuning, with production-grade Java examples.

---

## Table of Contents

- [Overview & Architecture](#overview--architecture)
- [Setup & Dependencies](#setup--dependencies)
- [Entity Mapping](#entity-mapping)
  - [Basic Mapping](#basic-mapping)
  - [Column Mapping](#column-mapping)
  - [Primary Key Strategies](#primary-key-strategies)
  - [Embedded Objects](#embedded-objects)
  - [Enums](#enums)
  - [Temporal & JSON Fields](#temporal--json-fields)
- [Relationships](#relationships)
  - [One-to-One](#one-to-one)
  - [One-to-Many / Many-to-One](#one-to-many--many-to-one)
  - [Many-to-Many](#many-to-many)
  - [Fetch Types](#fetch-types)
  - [Cascade Types](#cascade-types)
- [Repository Layer](#repository-layer)
  - [Repository Hierarchy](#repository-hierarchy)
  - [Derived Query Methods](#derived-query-methods)
  - [JPQL with @Query](#jpql-with-query)
  - [Native Queries](#native-queries)
  - [Projections](#projections)
  - [Specifications (Dynamic Queries)](#specifications-dynamic-queries)
  - [QueryDSL](#querydsl)
- [Pagination & Sorting](#pagination--sorting)
- [Transactions](#transactions)
- [Entity Lifecycle & Persistence Context](#entity-lifecycle--persistence-context)
  - [Entity States](#entity-states)
  - [Persistence Context (First-Level Cache)](#persistence-context-first-level-cache)
  - [Entity Lifecycle Callbacks](#entity-lifecycle-callbacks)
- [Auditing](#auditing)
- [Inheritance Mapping](#inheritance-mapping)
- [Performance & Common Pitfalls](#performance--common-pitfalls)
  - [N+1 Problem](#n1-problem)
  - [Fetch Join](#fetch-join)
  - [Batch Fetching](#batch-fetching)
  - [Second-Level Cache](#second-level-cache)
  - [Projections for Read Performance](#projections-for-read-performance)
  - [Bulk Operations](#bulk-operations)
- [Custom Repository Implementation](#custom-repository-implementation)
- [Testing](#testing)
- [Configuration Reference](#configuration-reference)
- [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Overview & Architecture

```
┌─────────────────────────────────────────────────────┐
│                   Your Application                   │
├─────────────────────────────────────────────────────┤
│              Spring Data JPA (Repository)            │  ← your interfaces
├─────────────────────────────────────────────────────┤
│           JPA (Java Persistence API)                 │  ← standard spec
├─────────────────────────────────────────────────────┤
│         Hibernate ORM (JPA Implementation)           │  ← default provider
├─────────────────────────────────────────────────────┤
│              JDBC / Connection Pool                  │
├─────────────────────────────────────────────────────┤
│              Database (PostgreSQL, MySQL…)            │
└─────────────────────────────────────────────────────┘
```

**Key components:**
- **JPA** — Specification (javax.persistence / jakarta.persistence)
- **Hibernate** — Default JPA implementation (ORM engine)
- **Spring Data JPA** — Abstraction layer over JPA; auto-generates repository implementations
- **EntityManager** — JPA's primary interface to the persistence context
- **Persistence Context** — First-level cache; tracks entity state within a transaction

---

## Setup & Dependencies

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>

<!-- Optional but recommended -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>
```

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: user
    password: secret
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2
      connection-timeout: 30000

  jpa:
    hibernate:
      ddl-auto: validate          # validate | update | create | create-drop | none
    show-sql: false               # use logging in prod instead
    properties:
      hibernate:
        format_sql: true
        default_batch_fetch_size: 100
        dialect: org.hibernate.dialect.PostgreSQLDialect

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql: TRACE  # logs bind parameters
```

> **`ddl-auto` guide:**
> - `validate` — prod (validate schema matches entities, fail fast)
> - `update` — dev only (risky in prod)
> - `none` — prod with Flyway/Liquibase migrations
> - `create-drop` — tests only

---

## Entity Mapping

### Basic Mapping

```java
import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(
    name = "users",
    indexes = {
        @Index(name = "idx_user_email", columnList = "email"),
        @Index(name = "idx_user_created", columnList = "created_at")
    },
    uniqueConstraints = @UniqueConstraint(name = "uk_user_email", columnNames = "email")
)
@Getter
@Setter
@NoArgsConstructor
@ToString(exclude = {"orders"}) // exclude collections to avoid lazy loading in toString
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @EqualsAndHashCode.Include
    private Long id;

    @Column(name = "first_name", nullable = false, length = 50)
    private String firstName;

    @Column(name = "last_name", nullable = false, length = 50)
    private String lastName;

    @Column(unique = true, nullable = false, length = 100)
    private String email;

    @Column(name = "is_active", nullable = false)
    private boolean active = true;

    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Order> orders = new ArrayList<>();
}
```

> ⚠️ **Lombok with JPA:**
> - Use `@ToString(exclude = {...})` for collections — avoids lazy loading exceptions
> - Use `@EqualsAndHashCode(onlyExplicitlyIncluded = true)` — never use mutable fields or collections
> - Avoid `@Data` on entities — it generates problematic `equals`/`hashCode`

---

### Column Mapping

```java
@Column(
    name = "description",       // DB column name
    nullable = false,           // NOT NULL constraint
    length = 500,               // VARCHAR(500)
    unique = false,
    insertable = true,
    updatable = false,          // immutable after insert (e.g., created_at)
    columnDefinition = "TEXT"   // override DDL type
)
private String description;

@Column(precision = 15, scale = 2) // DECIMAL(15,2)
private BigDecimal price;

@Lob                              // maps to TEXT / CLOB / BLOB
@Column(columnDefinition = "TEXT")
private String content;
```

---

### Primary Key Strategies

```java
// 1. IDENTITY — auto-increment (MySQL, PostgreSQL SERIAL)
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;

// 2. SEQUENCE — database sequence (best for batch inserts with PostgreSQL)
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "user_seq")
@SequenceGenerator(name = "user_seq", sequenceName = "user_id_seq", allocationSize = 50)
private Long id;

// 3. UUID — distributed systems, no DB roundtrip for ID
@Id
@GeneratedValue(strategy = GenerationType.UUID)
private UUID id;

// 4. Manual / Natural key
@Id
private String isoCode; // e.g., country codes "IN", "US"
```

> **SEQUENCE + allocationSize** is best for bulk inserts — Hibernate pre-allocates IDs in batches (size 50 means 1 DB call per 50 entities).

---

### Embedded Objects

```java
@Embeddable
@Getter
@NoArgsConstructor
@AllArgsConstructor
public class Address {
    @Column(name = "street", length = 200)
    private String street;

    @Column(name = "city", length = 100)
    private String city;

    @Column(name = "pin_code", length = 10)
    private String pinCode;

    @Column(name = "country", length = 50)
    private String country;
}

@Entity
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "street",  column = @Column(name = "billing_street")),
        @AttributeOverride(name = "city",    column = @Column(name = "billing_city")),
        @AttributeOverride(name = "pinCode", column = @Column(name = "billing_pin"))
    })
    private Address billingAddress;

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "street", column = @Column(name = "shipping_street")),
        @AttributeOverride(name = "city",   column = @Column(name = "shipping_city")),
        @AttributeOverride(name = "pinCode",column = @Column(name = "shipping_pin"))
    })
    private Address shippingAddress;
}
```

---

### Enums

```java
public enum OrderStatus {
    PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED
}

@Entity
public class Order {
    // ❌ Default ORDINAL — breaks if enum reordered
    @Enumerated(EnumType.ORDINAL)
    private OrderStatus status;

    // ✅ STRING — safe, readable
    @Enumerated(EnumType.STRING)
    @Column(name = "status", length = 20)
    private OrderStatus status;
}
```

---

### Temporal & JSON Fields

```java
@Entity
public class Article {

    // Java 8+ date/time — no @Temporal needed
    @Column(name = "published_at")
    private LocalDateTime publishedAt;

    @Column(name = "publish_date")
    private LocalDate publishDate;

    // Auto-set timestamps (see Auditing section for cleaner approach)
    @Column(name = "created_at", updatable = false)
    private Instant createdAt;

    @Column(name = "updated_at")
    private Instant updatedAt;

    @PrePersist
    void onCreate() { createdAt = updatedAt = Instant.now(); }

    @PreUpdate
    void onUpdate() { updatedAt = Instant.now(); }

    // JSON column (PostgreSQL jsonb / MySQL JSON)
    @Column(columnDefinition = "jsonb")
    @Convert(converter = MapToJsonConverter.class) // custom converter
    private Map<String, Object> metadata;
}

// Custom AttributeConverter for JSON
@Converter
public class MapToJsonConverter implements AttributeConverter<Map<String, Object>, String> {
    private static final ObjectMapper MAPPER = new ObjectMapper();

    @Override
    public String convertToDatabaseColumn(Map<String, Object> map) {
        try { return MAPPER.writeValueAsString(map); }
        catch (JsonProcessingException e) { throw new IllegalArgumentException(e); }
    }

    @Override
    public Map<String, Object> convertToEntityAttribute(String json) {
        try { return MAPPER.readValue(json, new TypeReference<>() {}); }
        catch (JsonProcessingException e) { throw new IllegalArgumentException(e); }
    }
}
```

---

## Relationships

### One-to-One

```java
@Entity
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // FK lives in user_profiles table (recommended — avoids nullable FK in users)
    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL,
              fetch = FetchType.LAZY, optional = false)
    private UserProfile profile;
}

@Entity
@Table(name = "user_profiles")
public class UserProfile {
    @Id
    private Long id; // same PK as user — shared primary key pattern

    @OneToOne(fetch = FetchType.LAZY)
    @MapsId // UserProfile.id = User.id
    @JoinColumn(name = "user_id")
    private User user;

    private String bio;
    private String avatarUrl;
}
```

> **`@MapsId`** (shared primary key) is the best One-to-One strategy — avoids an extra `user_id` column in `user_profiles` and guarantees `LAZY` loading actually works (no extra query needed to check if profile exists).

---

### One-to-Many / Many-to-One

```java
@Entity
public class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // OWNER side — FK lives here
    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    @OrderBy("createdAt DESC")
    private List<OrderItem> items = new ArrayList<>();

    // Bidirectional helper methods — keep both sides in sync
    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this);
    }

    public void removeItem(OrderItem item) {
        items.remove(item);
        item.setOrder(null);
    }
}

@Entity
@Table(name = "order_items")
public class OrderItem {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;

    @Column(nullable = false)
    private int quantity;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal unitPrice;
}
```

> ⚠️ **Always define `mappedBy`** on the inverse side (`@OneToMany`). The side with `@JoinColumn` is the owning side (controls the FK).

---

### Many-to-Many

```java
@Entity
public class Student {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    // Unidirectional many-to-many with join table
    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    @JoinTable(
        name = "student_courses",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>(); // Set avoids duplicates
}

@Entity
public class Course {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;

    @ManyToMany(mappedBy = "courses")
    private Set<Student> students = new HashSet<>();
}
```

**When the join table needs extra columns — use an explicit join entity:**

```java
@Entity
@Table(name = "student_courses")
public class StudentCourse {
    @EmbeddedId
    private StudentCourseId id = new StudentCourseId();

    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("studentId")
    private Student student;

    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("courseId")
    private Course course;

    @Column(name = "enrolled_at")
    private LocalDate enrolledAt;

    @Column(name = "grade")
    private String grade;
}

@Embeddable
public class StudentCourseId implements Serializable {
    private Long studentId;
    private Long courseId;
    // equals + hashCode required
}
```

---

### Fetch Types

| FetchType | Default on | Behavior |
|---|---|---|
| `LAZY` | `@OneToMany`, `@ManyToMany` | Load on first access (proxy) |
| `EAGER` | `@ManyToOne`, `@OneToOne` | Load immediately with parent |

```java
// ✅ Always use LAZY explicitly — avoid EAGER
@ManyToOne(fetch = FetchType.LAZY)
@OneToMany(fetch = FetchType.LAZY)
@OneToOne(fetch = FetchType.LAZY)  // requires @MapsId or byte-code enhancement for true lazy
```

> ⚠️ `EAGER` on collections causes N+1 and unexpected joins on every query. **Default everything to LAZY** and fetch eagerly only when needed via JOIN FETCH.

---

### Cascade Types

| CascadeType | Meaning |
|---|---|
| `PERSIST` | Saving parent saves new children |
| `MERGE` | Merging parent merges children |
| `REMOVE` | Deleting parent deletes children |
| `REFRESH` | Refreshing parent refreshes children |
| `DETACH` | Detaching parent detaches children |
| `ALL` | All of the above |

```java
// Parent → child lifecycle: use ALL + orphanRemoval
@OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)

// Cross-aggregate reference: only PERSIST + MERGE (no REMOVE)
@ManyToOne(cascade = {CascadeType.PERSIST, CascadeType.MERGE})

// Never cascade REMOVE on @ManyToMany — it deletes shared entities
@ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
```

---

## Repository Layer

### Repository Hierarchy

```
Repository<T, ID>                    ← marker interface
    └── CrudRepository<T, ID>        ← save, findById, findAll, delete, count
        └── PagingAndSortingRepository<T, ID>  ← findAll(Sort), findAll(Pageable)
            └── JpaRepository<T, ID> ← flush, saveAndFlush, deleteInBatch, getReferenceById
```

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    // Spring Data generates the implementation at startup
}
```

**Key `JpaRepository` methods:**

```java
// Save
User saved = userRepo.save(user);           // INSERT or UPDATE
userRepo.saveAll(List.of(u1, u2, u3));
userRepo.saveAndFlush(user);               // immediately flushes to DB

// Read
Optional<User> user = userRepo.findById(1L);
User ref = userRepo.getReferenceById(1L);  // lazy proxy — no DB hit until accessed
List<User> all = userRepo.findAll();

// Delete
userRepo.deleteById(1L);
userRepo.delete(user);
userRepo.deleteAllInBatch(users);           // single DELETE IN (...)

// Existence & count
boolean exists = userRepo.existsById(1L);
long count = userRepo.count();
```

---

### Derived Query Methods

Spring Data generates queries from method names:

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // SELECT * FROM users WHERE email = ?
    Optional<User> findByEmail(String email);

    // SELECT * FROM users WHERE first_name = ? AND active = ?
    List<User> findByFirstNameAndActive(String firstName, boolean active);

    // SELECT * FROM users WHERE last_name LIKE '%?%'
    List<User> findByLastNameContainingIgnoreCase(String lastName);

    // SELECT * FROM users WHERE created_at BETWEEN ? AND ?
    List<User> findByCreatedAtBetween(Instant from, Instant to);

    // SELECT * FROM users WHERE age >= ? ORDER BY last_name ASC
    List<User> findByAgeGreaterThanEqualOrderByLastNameAsc(int age);

    // SELECT COUNT(*) FROM users WHERE active = ?
    long countByActive(boolean active);

    // DELETE FROM users WHERE active = ?
    @Modifying
    @Transactional
    void deleteByActive(boolean active);

    // SELECT TOP 5 FROM users ORDER BY created_at DESC
    List<User> findTop5ByOrderByCreatedAtDesc();

    // EXISTS query
    boolean existsByEmail(String email);
}
```

**Keyword reference:**

| Keyword | SQL equivalent |
|---|---|
| `And` / `Or` | `AND` / `OR` |
| `Is`, `Equals` | `=` |
| `Between` | `BETWEEN ? AND ?` |
| `LessThan` / `GreaterThan` | `<` / `>` |
| `Like` / `NotLike` | `LIKE` / `NOT LIKE` |
| `StartingWith` / `EndingWith` / `Containing` | `LIKE 'x%'` / `'%x'` / `'%x%'` |
| `IgnoreCase` | `LOWER(col) = LOWER(?)` |
| `OrderBy…Asc/Desc` | `ORDER BY col ASC/DESC` |
| `Top N` / `First N` | `LIMIT N` |
| `In` / `NotIn` | `IN (...)` |
| `IsNull` / `IsNotNull` | `IS NULL` / `IS NOT NULL` |
| `True` / `False` | `= true` / `= false` |

---

### JPQL with @Query

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    // JPQL — works on entity/field names, not table/column names
    @Query("SELECT o FROM Order o WHERE o.status = :status AND o.user.id = :userId")
    List<Order> findByStatusAndUserId(@Param("status") OrderStatus status,
                                      @Param("userId") Long userId);

    // JOIN FETCH — prevents N+1 (loads items in single query)
    @Query("SELECT DISTINCT o FROM Order o " +
           "LEFT JOIN FETCH o.items i " +
           "WHERE o.user.id = :userId")
    List<Order> findWithItemsByUserId(@Param("userId") Long userId);

    // Aggregate
    @Query("SELECT SUM(oi.quantity * oi.unitPrice) FROM OrderItem oi WHERE oi.order.id = :orderId")
    Optional<BigDecimal> calculateOrderTotal(@Param("orderId") Long orderId);

    // UPDATE / DELETE — requires @Modifying
    @Modifying
    @Transactional
    @Query("UPDATE Order o SET o.status = :status WHERE o.id IN :ids")
    int bulkUpdateStatus(@Param("status") OrderStatus status, @Param("ids") List<Long> ids);

    // Pageable with @Query
    @Query(
        value = "SELECT o FROM Order o WHERE o.status = :status",
        countQuery = "SELECT COUNT(o) FROM Order o WHERE o.status = :status"
    )
    Page<Order> findByStatus(@Param("status") OrderStatus status, Pageable pageable);
}
```

---

### Native Queries

```java
// Use when JPQL can't express the query (window functions, CTEs, DB-specific features)
@Query(
    value = """
        SELECT u.*, 
               COUNT(o.id) AS order_count,
               SUM(o.total) AS lifetime_value
        FROM users u
        LEFT JOIN orders o ON o.user_id = u.id
        WHERE u.created_at > :since
        GROUP BY u.id
        HAVING COUNT(o.id) > :minOrders
        ORDER BY lifetime_value DESC
        LIMIT :limit
        """,
    nativeQuery = true
)
List<Object[]> findHighValueCustomers(@Param("since") Instant since,
                                       @Param("minOrders") int minOrders,
                                       @Param("limit") int limit);

// With projection interface (cleaner than Object[])
@Query(
    value = "SELECT id, email, first_name AS firstName FROM users WHERE active = true",
    nativeQuery = true
)
List<UserSummary> findActiveUserSummaries();
```

---

### Projections

Fetch only required columns — avoids loading full entities for read-only views.

**Interface projection (auto-proxied by Spring Data):**

```java
public interface UserSummary {
    Long getId();
    String getFirstName();
    String getLastName();
    String getEmail();

    // Computed projection
    @Value("#{target.firstName + ' ' + target.lastName}")
    String getFullName();
}

public interface UserRepository extends JpaRepository<User, Long> {
    List<UserSummary> findByActive(boolean active);

    // Generic projection
    <T> List<T> findByActive(boolean active, Class<T> type);
}
```

**Class-based (DTO) projection:**

```java
public record UserDto(Long id, String firstName, String email) {}

public interface UserRepository extends JpaRepository<User, Long> {
    @Query("SELECT new com.example.dto.UserDto(u.id, u.firstName, u.email) FROM User u WHERE u.active = true")
    List<UserDto> findActiveDtos();
}
```

**Dynamic projection:**

```java
// Same query, different return types
List<UserSummary> summaries = userRepo.findByActive(true, UserSummary.class);
List<UserDto>     dtos      = userRepo.findByActive(true, UserDto.class);
```

---

### Specifications (Dynamic Queries)

For complex, runtime-built predicates (search filters, admin screens).

```java
// 1. Add JpaSpecificationExecutor to repository
public interface UserRepository extends JpaRepository<User, Long>,
                                         JpaSpecificationExecutor<User> {}

// 2. Define specification factory
public final class UserSpecs {

    public static Specification<User> hasEmail(String email) {
        return (root, query, cb) ->
            email == null ? null : cb.equal(root.get("email"), email);
    }

    public static Specification<User> isActive(Boolean active) {
        return (root, query, cb) ->
            active == null ? null : cb.equal(root.get("active"), active);
    }

    public static Specification<User> createdAfter(Instant date) {
        return (root, query, cb) ->
            date == null ? null : cb.greaterThan(root.get("createdAt"), date);
    }

    public static Specification<User> hasNameLike(String name) {
        return (root, query, cb) ->
            name == null ? null :
            cb.like(cb.lower(root.get("firstName")), "%" + name.toLowerCase() + "%");
    }
}

// 3. Compose at runtime
Specification<User> spec = Specification
    .where(UserSpecs.isActive(true))
    .and(UserSpecs.createdAfter(Instant.now().minus(30, ChronoUnit.DAYS)))
    .and(UserSpecs.hasNameLike(searchTerm));

Page<User> results = userRepo.findAll(spec, PageRequest.of(0, 20));
```

---

### QueryDSL

Type-safe, fluent queries — alternative to Specifications for complex use cases.

```xml
<dependency>
    <groupId>com.querydsl</groupId>
    <artifactId>querydsl-jpa</artifactId>
    <classifier>jakarta</classifier>
</dependency>
```

```java
// Q-classes auto-generated by APT plugin (QUser, QOrder, etc.)
public interface UserRepository extends JpaRepository<User, Long>,
                                         QuerydslPredicateExecutor<User> {}

// Usage
QUser user = QUser.user;
Predicate predicate = user.active.isTrue()
    .and(user.createdAt.after(Instant.now().minus(30, ChronoUnit.DAYS)))
    .and(user.email.containsIgnoreCase("@gmail.com"));

Iterable<User> users = userRepo.findAll(predicate);
```

---

## Pagination & Sorting

```java
// Pageable — page number (0-based), size, sort
Pageable pageable = PageRequest.of(0, 20, Sort.by("lastName").ascending()
                                               .and(Sort.by("firstName").ascending()));

// Page — includes total count (2 queries)
Page<User> page = userRepo.findAll(pageable);

int     totalPages   = page.getTotalPages();
long    totalElements = page.getTotalElements();
List<User> content   = page.getContent();
boolean hasNext      = page.hasNext();
boolean hasPrevious  = page.hasPrevious();

// Slice — no total count (1 query, good for infinite scroll)
Slice<User> slice = userRepo.findByActive(true, pageable);
boolean hasMoreData = slice.hasNext();

// Sort only
List<User> sorted = userRepo.findAll(Sort.by(Sort.Direction.DESC, "createdAt"));

// Cursor-based pagination (no offset, better performance at scale)
@Query("SELECT u FROM User u WHERE u.id > :lastId ORDER BY u.id ASC LIMIT :limit")
List<User> findNextPage(@Param("lastId") Long lastId, @Param("limit") int limit);
```

---

## Transactions

```java
@Service
public class OrderService {

    private final OrderRepository orderRepo;
    private final UserRepository  userRepo;

    // Basic — commits on success, rolls back on RuntimeException
    @Transactional
    public Order placeOrder(Long userId, OrderRequest request) {
        User user = userRepo.findById(userId)
            .orElseThrow(() -> new EntityNotFoundException("User not found: " + userId));

        Order order = new Order();
        order.setUser(user);
        request.getItems().forEach(item -> order.addItem(mapToItem(item)));
        return orderRepo.save(order);
    }

    // Read-only — Hibernate skips dirty checking (performance gain)
    @Transactional(readOnly = true)
    public List<Order> getUserOrders(Long userId) {
        return orderRepo.findByUserId(userId);
    }

    // Checked exceptions need explicit rollbackFor
    @Transactional(rollbackFor = {PaymentException.class, InventoryException.class})
    public void processPayment(Long orderId) throws PaymentException {
        // ...
    }

    // Requires its own transaction regardless of caller
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveAuditLog(AuditEntry entry) {
        auditRepo.save(entry); // always committed independently
    }

    // Serializable — for critical operations
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public void transferFunds(Long fromId, Long toId, BigDecimal amount) {
        // ...
    }
}
```

> **`readOnly = true`** benefits:
> 1. Hibernate skips dirty checking on flush
> 2. Some JDBC drivers route to read replicas
> 3. Snapshot isolation in PostgreSQL uses shared locks

---

## Entity Lifecycle & Persistence Context

### Entity States

```
New / Transient ──── persist() ──→ Managed
                                      │
                                   flush() → SQL sent to DB
                                      │
                                   commit() → committed
                                      │
Detached ←─────── detach() / close() ┘
    │
  merge() → Managed (re-attached copy)

Removed ←─── remove() ── Managed
    │
 commit() → deleted from DB
```

```java
// Transient — not tracked
User user = new User();
user.setEmail("a@b.com");

// Managed — tracked by persistence context
User managed = entityManager.persist(user);  // or repo.save(user)

// Detached — changes NOT tracked
entityManager.detach(managed);
User reattached = entityManager.merge(managed); // back to managed (creates new instance)

// Removed — scheduled for DELETE
entityManager.remove(managed);

// getReferenceById — lazy proxy (no SELECT until field accessed)
User proxy = userRepo.getReferenceById(1L); // useful for setting FK without loading entity
order.setUser(proxy); // just sets the FK, no extra query
```

---

### Persistence Context (First-Level Cache)

```java
@Transactional
public void demo() {
    User u1 = userRepo.findById(1L).get(); // SELECT
    User u2 = userRepo.findById(1L).get(); // NO SELECT — returns cached instance

    System.out.println(u1 == u2); // true — same object reference

    u1.setFirstName("Updated");
    // No explicit save() needed — dirty checking detects the change
    // Hibernate sends UPDATE on flush (before commit)
}
```

**Dirty checking:**
- At flush, Hibernate compares each managed entity's current state to its snapshot
- Generates `UPDATE` for changed fields automatically
- This is why you often don't call `save()` on managed entities

**Flushing:**
```java
// FlushModeType.AUTO (default) — flushes before queries and on commit
// FlushModeType.COMMIT — flushes only on commit

entityManager.flush();       // explicit flush — forces SQL to DB within transaction
entityManager.clear();       // clears first-level cache (use after bulk ops)
entityManager.refresh(user); // reload entity from DB, discarding unsaved changes
```

---

### Entity Lifecycle Callbacks

```java
@Entity
public class Order {

    @PrePersist
    void beforeCreate() {
        this.createdAt = Instant.now();
        this.status = OrderStatus.PENDING;
    }

    @PostPersist
    void afterCreate() {
        // e.g., publish domain event
    }

    @PreUpdate
    void beforeUpdate() {
        this.updatedAt = Instant.now();
    }

    @PostUpdate
    void afterUpdate() { }

    @PreRemove
    void beforeDelete() {
        // validation, logging
    }

    @PostRemove
    void afterDelete() { }

    @PostLoad
    void afterLoad() {
        // computed fields after loading
    }
}
```

---

## Auditing

```java
// 1. Enable JPA Auditing
@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
public class JpaConfig {

    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext())
            .map(ctx -> ctx.getAuthentication())
            .filter(auth -> auth != null && auth.isAuthenticated())
            .map(auth -> auth.getName());
    }
}

// 2. Base auditable entity — extend all entities from this
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
@Getter
public abstract class Auditable {

    @CreatedDate
    @Column(name = "created_at", updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    @Column(name = "updated_at")
    private Instant updatedAt;

    @CreatedBy
    @Column(name = "created_by", updatable = false, length = 100)
    private String createdBy;

    @LastModifiedBy
    @Column(name = "updated_by", length = 100)
    private String updatedBy;
}

// 3. Entity extends Auditable
@Entity
public class Product extends Auditable {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
}
```

---

## Inheritance Mapping

### Single Table (SINGLE_TABLE) — default, best performance

```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "payment_type", discriminatorType = DiscriminatorType.STRING)
public abstract class Payment {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private BigDecimal amount;
}

@Entity
@DiscriminatorValue("CREDIT_CARD")
public class CreditCardPayment extends Payment {
    private String cardNumber;
    private String expiryDate;
}

@Entity
@DiscriminatorValue("UPI")
public class UpiPayment extends Payment {
    private String upiId;
}
```

One table, nullable columns for subtype fields. Best for polymorphic queries.

---

### Joined Table (JOINED) — normalized, use for complex hierarchies

```java
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
public abstract class Vehicle {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String manufacturer;
}

@Entity
@Table(name = "cars")
public class Car extends Vehicle {
    private int numberOfDoors;
}

@Entity
@Table(name = "trucks")
public class Truck extends Vehicle {
    private double payloadCapacity;
}
```

Separate table per class with FK to parent. Normalized but requires JOINs.

---

### Table Per Class (TABLE_PER_CLASS) — avoid in most cases

```java
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class Animal { /* all columns duplicated in subclass tables */ }
```

Polymorphic queries use `UNION ALL` — poor performance. Rarely the right choice.

---

### Strategy Comparison

| Strategy | Tables | Polymorphic Query | Pros | Cons |
|---|---|---|---|---|
| `SINGLE_TABLE` | 1 | Simple | Best performance | Nullable columns, discriminator col |
| `JOINED` | 1 per class | JOIN | Normalized | JOIN overhead |
| `TABLE_PER_CLASS` | 1 per concrete class | UNION ALL | No NULLs | Worst query performance |

---

## Performance & Common Pitfalls

### N+1 Problem

```java
// ❌ N+1 — loads orders (1 query), then for each order loads items (N queries)
@Transactional(readOnly = true)
public void processOrders() {
    List<Order> orders = orderRepo.findAll(); // 1 query
    for (Order order : orders) {
        System.out.println(order.getItems().size()); // N queries!
    }
}
```

**Detect N+1:**
```yaml
# Enable SQL logging and count queries
logging.level.org.hibernate.SQL: DEBUG
# Or use Hibernate Statistics:
spring.jpa.properties.hibernate.generate_statistics: true
# Or use p6spy / datasource-proxy for query counting in tests
```

---

### Fetch Join

```java
// ✅ JOIN FETCH — single query loads orders + items
@Query("SELECT DISTINCT o FROM Order o LEFT JOIN FETCH o.items WHERE o.user.id = :userId")
List<Order> findWithItems(@Param("userId") Long userId);

// ✅ EntityGraph — alternative to JOIN FETCH, declarative
@EntityGraph(attributePaths = {"items", "items.product"})
List<Order> findByUserId(Long userId);

// ✅ Named EntityGraph on entity
@Entity
@NamedEntityGraph(
    name = "Order.withItems",
    attributeNodes = @NamedAttributeNode(value = "items", subgraph = "items.product"),
    subgraphs = @NamedSubgraph(name = "items.product", attributeNodes = @NamedAttributeNode("product"))
)
public class Order { /* ... */ }

// Repository usage
@EntityGraph("Order.withItems")
List<Order> findByStatus(OrderStatus status);
```

> ⚠️ **JOIN FETCH + Pagination = HHH90003004 warning** — Hibernate can't apply LIMIT at DB level when fetching collections. Fix: fetch IDs first, then load with JOIN FETCH:

```java
// Two-query pattern for safe paginated collection fetching
@Query("SELECT o.id FROM Order o WHERE o.status = :status")
Page<Long> findIdsByStatus(@Param("status") OrderStatus status, Pageable pageable);

@Query("SELECT DISTINCT o FROM Order o LEFT JOIN FETCH o.items WHERE o.id IN :ids")
List<Order> findWithItemsByIds(@Param("ids") List<Long> ids);
```

---

### Batch Fetching

```java
# application.yml — Hibernate fetches collections in batches of 100
spring.jpa.properties.hibernate.default_batch_fetch_size: 100
```

With batch fetching, loading `N` orders' items sends `ceil(N/100)` queries instead of N.

---

### Second-Level Cache

```xml
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-jcache</artifactId>
</dependency>
<dependency>
    <groupId>org.ehcache</groupId>
    <artifactId>ehcache</artifactId>
    <classifier>jakarta</classifier>
</dependency>
```

```yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          use_query_cache: true
          region.factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
```

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE) // or NONSTRICT_READ_WRITE, READ_ONLY
public class Product {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    // ...
}

// Cache a query result
@QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
List<Product> findByCategory(String category);
```

**Cache strategies:**
| Strategy | Use when |
|---|---|
| `READ_ONLY` | Immutable data (reference tables, config) |
| `NONSTRICT_READ_WRITE` | Rarely updated, stale reads acceptable |
| `READ_WRITE` | Frequently read, occasionally written |
| `TRANSACTIONAL` | High consistency required (JTA only) |

---

### Projections for Read Performance

```java
// ❌ Loading full entity for a list view
List<User> users = userRepo.findAll(); // loads all columns including blobs

// ✅ DTO projection — only fetches needed columns
@Query("SELECT new com.example.dto.UserListDto(u.id, u.firstName, u.email) FROM User u")
List<UserListDto> findAllForList();
```

---

### Bulk Operations

```java
// Bulk UPDATE — bypasses entity lifecycle (no dirty checking, no callbacks)
@Modifying
@Transactional
@Query("UPDATE Product p SET p.price = p.price * 1.1 WHERE p.category = :cat")
int increasePriceByCategory(@Param("cat") String category);

// After bulk op, clear persistence context (stale first-level cache)
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Transactional
@Query("UPDATE User u SET u.active = false WHERE u.lastLoginAt < :cutoff")
int deactivateInactiveUsers(@Param("cutoff") Instant cutoff);

// Bulk INSERT with JDBC batch (Spring Data JPA saveAll triggers batching if sequence strategy used)
userRepo.saveAll(largeListOfUsers); // batched if spring.jpa.properties.hibernate.jdbc.batch_size set

// application.yml
spring.jpa.properties.hibernate.jdbc.batch_size: 50
spring.jpa.properties.hibernate.order_inserts: true
spring.jpa.properties.hibernate.order_updates: true
```

---

## Custom Repository Implementation

```java
// 1. Custom interface
public interface UserRepositoryCustom {
    List<User> searchUsers(UserSearchCriteria criteria);
}

// 2. Implementation — naming convention: {RepositoryName}Impl
@Repository
public class UserRepositoryImpl implements UserRepositoryCustom {

    @PersistenceContext
    private EntityManager em;

    @Override
    public List<User> searchUsers(UserSearchCriteria criteria) {
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<User> cq = cb.createQuery(User.class);
        Root<User> root = cq.from(User.class);

        List<Predicate> predicates = new ArrayList<>();

        if (criteria.getName() != null) {
            predicates.add(cb.like(cb.lower(root.get("firstName")),
                                   "%" + criteria.getName().toLowerCase() + "%"));
        }
        if (criteria.getActive() != null) {
            predicates.add(cb.equal(root.get("active"), criteria.getActive()));
        }

        cq.where(predicates.toArray(new Predicate[0]));
        cq.orderBy(cb.desc(root.get("createdAt")));

        return em.createQuery(cq)
                 .setMaxResults(criteria.getLimit())
                 .getResultList();
    }
}

// 3. Main repository extends both
public interface UserRepository extends JpaRepository<User, Long>,
                                         UserRepositoryCustom {}
```

---

## Testing

```java
// @DataJpaTest — loads only JPA slice (no web, no service layer)
// Uses in-memory H2 by default; each test method wraps in a transaction that rolls back
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE) // use real DB
@Testcontainers
class UserRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16")
        .withDatabaseName("testdb");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url",      postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private UserRepository userRepo;

    @Autowired
    private TestEntityManager tem; // test-scoped EntityManager

    @Test
    void shouldFindUserByEmail() {
        // Arrange
        User user = new User();
        user.setEmail("test@example.com");
        user.setFirstName("Test");
        user.setLastName("User");
        tem.persistAndFlush(user);
        tem.clear(); // clear cache to ensure actual DB read

        // Act
        Optional<User> found = userRepo.findByEmail("test@example.com");

        // Assert
        assertThat(found).isPresent();
        assertThat(found.get().getFirstName()).isEqualTo("Test");
    }

    @Test
    void shouldDetectNPlusOne() {
        // Using datasource-proxy or Hibernate statistics to assert query count
        Statistics stats = em.unwrap(Session.class).getSessionFactory().getStatistics();
        stats.setStatisticsEnabled(true);
        stats.clear();

        userRepo.findAll(); // your query under test

        assertThat(stats.getQueryExecutionCount()).isEqualTo(1); // assert no N+1
    }
}
```

---

## Configuration Reference

```yaml
spring:
  jpa:
    # DDL strategy
    hibernate:
      ddl-auto: validate

    # Show SQL (use logging in prod)
    show-sql: false

    # Open-in-view: always false in production
    open-in-view: false   # ← avoids lazy loading across HTTP request lifecycle

    properties:
      hibernate:
        # Dialect
        dialect: org.hibernate.dialect.PostgreSQLDialect

        # SQL formatting for logs
        format_sql: true

        # Batching
        jdbc.batch_size: 50
        order_inserts: true
        order_updates: true

        # Fetch optimization
        default_batch_fetch_size: 100

        # Statistics (disable in prod)
        generate_statistics: false

        # Second-level cache
        cache.use_second_level_cache: false
        cache.use_query_cache: false

  # HikariCP connection pool
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
      leak-detection-threshold: 60000
```

> ⚠️ **`open-in-view: false` is critical in production.** When `true` (Spring Boot default), the EntityManager stays open for the entire HTTP request, allowing lazy loading in view templates. This hides N+1 problems and ties a DB connection to the HTTP thread for the full request duration.

---

## Quick Reference Cheat Sheet

### Annotations

| Annotation | Purpose |
|---|---|
| `@Entity` | Marks class as JPA entity |
| `@Table` | Customize table name, indexes, constraints |
| `@Id` | Primary key |
| `@GeneratedValue` | PK generation strategy |
| `@Column` | Column mapping |
| `@Transient` | Exclude field from persistence |
| `@Embedded` / `@Embeddable` | Value object / composed type |
| `@OneToOne` / `@OneToMany` / `@ManyToOne` / `@ManyToMany` | Relationships |
| `@JoinColumn` | FK column definition |
| `@MappedSuperclass` | Base class — not an entity itself |
| `@Inheritance` | Inheritance mapping strategy |
| `@EntityListeners` | Register lifecycle listener |
| `@Version` | Optimistic locking version field |
| `@Lock(LockModeType.PESSIMISTIC_WRITE)` | Pessimistic lock |
| `@QueryHint` | Hints to JPA provider |
| `@NamedEntityGraph` | Declare fetch graph on entity |

### Optimistic Locking

```java
@Entity
public class Product {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Version   // Hibernate auto-increments on each UPDATE
    private Long version; // OptimisticLockException if concurrent update detected
}
```

### Pessimistic Locking

```java
@Lock(LockModeType.PESSIMISTIC_WRITE) // SELECT ... FOR UPDATE
Optional<Account> findById(Long id);
```

### Common Mistakes Summary

| Mistake | Fix |
|---|---|
| EAGER fetch on collections | Always `FetchType.LAZY` |
| N+1 on lazy collections | `JOIN FETCH` or `@EntityGraph` |
| `open-in-view: true` in prod | Set to `false` |
| `@Transactional` on private methods | Use public methods only |
| Cascade REMOVE on `@ManyToMany` | Never; use only `PERSIST` + `MERGE` |
| `toString()`/`equals()` on collections | Exclude with `@ToString(exclude={})` |
| Paginating with JOIN FETCH | Fetch IDs first, then JOIN FETCH by IDs |
| `@Modifying` without `clearAutomatically` | Use `clearAutomatically = true` |
| Checked exception without `rollbackFor` | Add `rollbackFor` to `@Transactional` |
| Using `@Data` on entities | Use `@Getter @Setter` + explicit `equals`/`hashCode` |

---

*References: Spring Data JPA Docs (docs.spring.io) | Hibernate ORM Docs (hibernate.org) | Vlad Mihalcea — High-Performance Java Persistence*
