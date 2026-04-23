# Spring Boot — Complete Notes & Cheat Sheet
> My personal bible for Spring Boot mastery 🚀
> Author: H Mahabalaraju | Updated: 2026

---

## 1. What is Spring Boot?

Spring Boot is a Java framework built on top of Spring that simplifies application development.

- Eliminates boilerplate code with **auto-configuration**
- Comes with an **embedded Tomcat server** — no separate server setup needed
- Easy creation of **JAR, WAR, EAR** files — just run!
- Easy **dependency management** via starters
- Main goal → reduce development, unit test, and integration test time

---

## 2. Dependency Injection (DI)
Dependency injection is the fundamental aspect in the springframework through which the spring container "injects" objects into the other objects.

Simplifies object creation and management, promoting **loose coupling**.

Instead of creating objects manually:
```java
// Without DI — tight coupling
UserService service = new UserService(new UserRepository());

// With DI — Spring creates and injects automatically
@Autowired
private UserService service;
```

---

## 3. Spring IoC (Inversion of Control)

The **IoC Container** is the core of Spring Framework.

**Responsibilities:**
- Creates, manages, and configures application objects (beans)
- Automatically injects required dependencies into beans
- Manages bean lifecycle — creation, initialization, destruction
- Supports XML, annotation-based, and Java-based configuration
- Separates object creation from business logic → more flexible and testable

### Types of IoC Containers:

| Container | Description |
|---|---|
| **BeanFactory** | Basic container. Lazy initialization. Lightweight. |
| **ApplicationContext** | Advanced container. Eager initialization. Recommended for most apps. |

---

## 4. Coupling

Refers to the degree of direct knowledge one element has of another.
How often does a change in class A force related changes in class B?

### Tight Coupling ❌
Two classes are directly dependent — change one, must change the other.

```java
// Tight coupling — UserService directly depends on MySQLDatabase
public class UserService {
    MySQLDatabase db = new MySQLDatabase(); // tightly coupled!
}
```

### Loose Coupling ✅
Two objects are independent of each other.
In Spring — achieved using **Dependency Injection with interfaces (POJO model)**

```java
// Contract — depend on interface, not implementation
public interface Topic {
    void understand();
}

public class SpringTopic implements Topic {
    public void understand() {
        System.out.println("Learning Spring IoC: Loose Coupling!");
    }
}

public class KafkaTopic implements Topic {
    public void understand() {
        System.out.println("Learning Kafka: Loose Coupling!");
    }
}

// Now inject any implementation without changing the class
@Autowired
private Topic topic; // Spring decides which implementation to inject
```

---

## 5. Spring Annotations Cheat Sheet

| Annotation | Purpose |
|---|---|
| `@SpringBootApplication` | Main class — combines @Configuration + @EnableAutoConfiguration + @ComponentScan |
| `@RestController` | Marks class as REST controller |
| `@Controller` | Marks class as MVC controller |
| `@Service` | Business logic layer |
| `@Repository` | Data access layer |
| `@Component` | Generic Spring bean |
| `@Autowired` | Dependency injection |
| `@Bean` | Declares a bean in configuration class |
| `@Configuration` | Marks class as configuration |
| `@Value` | Injects values from properties file |
| `@RequestMapping` | Maps HTTP requests to handler methods |
| `@GetMapping` | HTTP GET |
| `@PostMapping` | HTTP POST |
| `@PutMapping` | HTTP PUT |
| `@DeleteMapping` | HTTP DELETE |
| `@PathVariable` | Extracts value from URL path |
| `@RequestBody` | Binds HTTP request body to method parameter |
| `@ResponseBody` | Binds method return value to HTTP response body |

---

## 6. Spring Boot Starters

| Starter | Purpose |
|---|---|
| `spring-boot-starter-web` | REST APIs, Spring MVC, Embedded Tomcat |
| `spring-boot-starter-data-jpa` | JPA, Hibernate, Database |
| `spring-boot-starter-security` | Spring Security, OAuth2 |
| `spring-boot-starter-kafka` | Apache Kafka |
| `spring-boot-starter-test` | JUnit, Mockito |
| `spring-boot-starter-actuator` | Monitoring and health checks |
| `spring-boot-starter-cache` | Caching support |

---

## 7. Bean Lifecycle

```
Spring Container Starts
        ↓
Bean Definition Loaded
        ↓
Bean Instantiated
        ↓
Dependencies Injected
        ↓
@PostConstruct called
        ↓
Bean Ready to Use
        ↓
@PreDestroy called
        ↓
Bean Destroyed
```

---

## 8. Bean Scopes

| Scope | Description |
|---|---|
| `singleton` | Default. One instance per Spring container |
| `prototype` | New instance every time bean is requested |
| `request` | One instance per HTTP request |
| `session` | One instance per HTTP session |

```java
@Bean
@Scope("prototype")
public UserService userService() {
    return new UserService();
}
```

---

## 9. application.properties vs application.yml

```properties
# application.properties
server.port=8080
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=root
spring.datasource.password=password
```

```yaml
# application.yml
server:
  port: 8080
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    username: root
    password: password
```

---

## 10. REST API Example

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping
    public List<User> getAllUsers() {
        return userService.findAll();
    }

    @GetMapping("/{id}")
    public User getUserById(@PathVariable Long id) {
        return userService.findById(id);
    }

    @PostMapping
    public User createUser(@RequestBody User user) {
        return userService.save(user);
    }

    @PutMapping("/{id}")
    public User updateUser(@PathVariable Long id, @RequestBody User user) {
        return userService.update(id, user);
    }

    @DeleteMapping("/{id}")
    public void deleteUser(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

---

## 11. Exception Handling

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<String> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(ex.getMessage());
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<String> handleGeneral(Exception ex) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body("Something went wrong");
    }
}
```

---

## 12. Key Interview Questions

**Q: What is Spring Boot auto-configuration?**
A: Spring Boot automatically configures your application based on dependencies present in classpath. No manual configuration needed.

**Q: Difference between @Component, @Service, @Repository?**
A: All are Spring beans. @Service = business layer, @Repository = data layer (also handles DB exceptions), @Component = generic.

**Q: What is @SpringBootApplication?**
A: Combination of @Configuration + @EnableAutoConfiguration + @ComponentScan.

**Q: Difference between BeanFactory and ApplicationContext?**
A: BeanFactory is basic, lazy. ApplicationContext is advanced, eager, supports i18n, events, AOP. Always prefer ApplicationContext.

**Q: What is IoC?**
A: Inversion of Control — Spring container controls object creation instead of developer. Promotes loose coupling.

**Q: What is the difference between tight and loose coupling?**
A: Tight = classes directly dependent, hard to test. Loose = classes depend on interfaces, easy to swap implementations, highly testable.

---

## 13. My Learning Progress

- [x] Spring Boot basics
- [x] IoC and DI
- [x] Tight vs Loose coupling
- [ ] Spring Security & OAuth 2.0
- [ ] Spring Data JPA & Hibernate
- [ ] Spring Boot Caching
- [ ] Spring Boot + Kafka
- [ ] Microservices with Spring Boot
- [ ] Docker + Spring Boot
- [ ] AWS + Spring Boot

---

*"Strong engineers never stop learning"* — Mahabalaraju H
