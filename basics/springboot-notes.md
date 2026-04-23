# Spring Boot — Complete Notes & Cheat Sheet
> My personal notes for Spring Boot mastery 
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
Dependency Injection (DI) is a design pattern where the Spring container (ApplicationContext) creates objects and injects their dependencies in the other class.

Simplifies object creation and management, promoting **loose coupling**.

Instead of creating objects manually:
```java
// Without DI — tight coupling
UserService service = new UserService();

// With DI — Spring creates and injects automatically
@Autowired
private UserService service;
```
### Types of Dependency Injection

1️⃣ Constructor Injection
The dependencies are passed through the class constructor. This is the best practice recommended by the Spring team
```java
@Component
public class UserService {

    private final OrderService orderService;

    public UserService(OrderService orderService) {
        this.orderService = orderService;
    }
}

using lombok 

@Service
@RequiredArgsConstructor // Automatically creates the constructor
public class EmailService {
    private final MailSender mailSender; // Lombok will inject this
}
```
2️⃣ Setter Injection
The dependency is provided via a public "setter" method after the object is created.
```java
@Component
public class UserService {

    private OrderService orderService;

    @Autowired
    public void setOrderService(OrderService orderService) {
        this.orderService = orderService;
    }
}
```

3️⃣ Field Injection

You put @Autowired directly on the private variable. No constructor or setter needed.

```java
@Component
public class UserService {

    @Autowired
    private OrderService orderService;
}
```
##  Dependency Injection Types Comparison

| Feature | Constructor Injection | Setter Injection | Field Injection |
|--------|---------------------|------------------|----------------|
| Immutability |  Yes (can use `final`) |  No |  No |
| Circular Dependency |  Not supported (fails fast) |  Possible (not recommended) |  Possible (not recommended) |
| Testing | Easiest | Moderate |  Hard |
| Recommendation |  Best practice | Use for optional deps | Avoid |

---

---

## 3. Spring IoC (Inversion of Control)

The **IoC Container** is a principle in software engineering in which we are transfering the control of objects to the container

**Responsibilities:**
- Manages bean lifecycle — creation, initialization, destruction
- Automatically injects required dependencies into beans
- Supports XML, annotation-based, and Java-based configuration
- Separates object creation from business logic → more flexible and testable

  the spring IoC container makes use of java pojo class and configuration metadata to produce a fully configured and executable system or application.

### Types of IoC Containers:

| Container | Description |
|---|---|
| **BeanFactory** | Basic container. Lazy initialization. Lightweight. |
| **ApplicationContext** | Advanced container. Eager initialization. Recommended for most apps. |

ApplicationContext is built on the top of bean factory interface. It adds some extra functionality like AOP , event propogation ,Internationalization.., its recommended over BeanFactory.

---

## 4. Coupling

Refers to the degree of direct knowledge one element has of another.
How often does a change in class A force related changes in class B?

### Tight Coupling ❌
Two classes are directly dependent — change one, must change the other.

```java
UserService service = new UserService(new UserRepository());
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
##  Bean defination: 
 The Bean is an object thats instantiated , assembled and managed by a spring IoC container. 
 ### Pojo class : 
 plain old java objects : It's an ordinary java object it seperates the business logics from the model class not bound by any special restriction.
 its a template for Bean defination 
 
##  Bean Scopes

| Scope | Description |
|---|---|
| `singleton` | Default. One instance per Spring container |
| `prototype` | New instance every time bean is requested |
| `request` | One instance per HTTP request |
| `session` | One instance per HTTP session |



# Singleton: The Spring IoC container creates exactly one instance of the bean.
Every time you @Autowired it or call context.getBean(), Spring gives you a reference to that same original object.

Example :

@Component
public class UserService {
}
@SpringBootApplication
public class ApiGateway1Application {
	public static void main(String[] args) {
	ConfigurableApplicationContext context = SpringApplication.run(ApiGateway1Application.class, args);
		UserService obj1 = context.getBean(UserService.class);
		UserService obj2 = context.getBean(UserService.class);
		
		 System.out.println(obj1 == obj2);
	        System.out.println(obj2);

}

# prototype: A new instance is created every single time the bean is requested.

@Bean
@Scope("prototype")
public UserService userService() {
    return new UserService();
}

## 🔹 Core Differences

| Feature | Singleton Scope | Prototype Scope |
|--------|----------------|----------------|
| Instance Count | One instance per Spring IoC container | New instance every request |
| Creation Time | Eager (created at startup by default) | Lazy (created on demand) |
| State | Typically stateless | Typically stateful |
| Destruction | Managed by Spring (`@PreDestroy` works) | Not managed by Spring |
| Use Case | Service, Repository, Controller | User-specific / temporary objects |

---
## 7. Bean Lifecycle

```
Spring Container Starts
        ↓
Bean Definitions Loaded
        ↓
Bean Instantiated (Constructor)
        ↓
Dependencies Injected (@Autowired)
        ↓
@PostConstruct called (init)
        ↓
Bean Ready to Use
        ↓
Container Shutdown
        ↓
@PreDestroy called (destroy)
        ↓
Bean Destroyed

```
Note:
- Dependencies are created and initialized BEFORE the dependent bean.
- During destruction, dependent beans are destroyed FIRST, then dependencies.
---


## Spring Bean Annotations

### @Configuration
- Is a Bean Factory
- Exists only to create, configure and hand over objects to Spring
- Has NO business logic of its own

### @Bean
- This METHOD produces a bean (used inside @Configuration)
- Real-time example: RestTemplate (HTTP client), DataSource (DB connection pool),
  third-party libraries (Jackson, Kafka, Redis) — where YOU control the construction
- Created ONCE, reused everywhere (Singleton by default)

### @Component
- Auto-register YOUR class as a bean
- Example: Utility classes, Helper classes

### Simple Rule
Your own class needed as bean     →  @Component
You need to CREATE other beans    →  @Configuration + @Bean

---
## 5. Spring Annotations Cheat Sheet

| Annotation | Purpose |
|---|---|
| `@SpringBootApplication` | Main class — combines @Configuration + @EnableAutoConfiguration + @ComponentScan |
| `@RestController` | Marks class as REST controller | we can't pass Html and @RestController = @Controller + @ResponseBody
| `@Controller` | Marks class as MVC controller | we can pass Html ,thymleaf ,Jsp | To return JSON, you must add @ResponseBody explicitly
| `@Service` | Business logic layer |
| `@Repository` | Data access layer |
| `@Component` | Generic Spring bean | Auto-register YOUR class as a bean 
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
