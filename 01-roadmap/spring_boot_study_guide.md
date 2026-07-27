# Spring Boot Senior Developer Interview Preparation Guide

This guide covers advanced Spring Boot architecture, Dependency Injection, Security, JPA/Hibernate persistence optimization, Resilience patterns, and Spring Cloud, tailored for senior product engineering interviews.

---

## 1. Application Lifecycle, Entrypoint & Layered Architecture

### Lifecycle & Entrypoint
* **Bootstrap Entrypoint:** Every Spring Boot application starts with a class annotated with `@SpringBootApplication`, which is a convenience annotation combining:
  * `@SpringBootConfiguration`: Declares the class as a source of bean definitions.
  * `@EnableAutoConfiguration`: Tells Spring Boot to automatically configure beans based on classpath dependencies (via entries in `META-INF/spring.factories` or `org.springframework.boot.autoconfigure.AutoConfiguration.imports` in Spring Boot 3).
  * `@ComponentScan`: Scans the package of the entrypoint class and all its subpackages to register beans.
* **Initialization Phases:**
  1. **Application Context Creation:** Sets up the environment, processes profiles, and initializes the Bean Factory.
  2. **Bean Definition Loading:** Scans the classpath, registers component classes, and loads XML/Java configurations.
  3. **Bean Post-Processing:** Instantiates beans, performs dependency injection, and runs custom bean post-processors.
  4. **Runners Execution:** Runs classes implementing `CommandLineRunner` or `ApplicationRunner` after the context is fully refreshed.

### Layered Architecture

```
[ Client Request ]
       │
       ▼
 [ Controller ]  (Routing, HTTP Status Codes, Input Validation via @Valid)
       │
       ▼
  [ Service ]    (Business Logic, Transaction Boundaries via @Transactional)
       │
       ▼
[ Repository ]   (Data Access / DAO Layer, Spring Data JPA / Hibernate)
       │
       ▼
 [ Database ]
```

---

## 2. Dependency Injection, IoC, & Core Annotations

### Inversion of Control (IoC) & Dependency Injection (DI)
* **IoC Container:** The core engine (`ApplicationContext`) that manages the lifecycle, configuration, and dependency resolution of Spring Beans.
* **Dependency Injection Types:**
  * **Constructor Injection (Recommended):** Guarantees dependencies are non-null (`final`), enables immutability, and simplifies unit testing (no reflection needed).
  * **Setter Injection:** Used for optional or mutable dependencies.
  * **Field Injection (`@Autowired` on private field):** Highly discouraged for senior developers. It hides dependencies, couples the class tightly to the Spring container, and makes unit testing difficult without reflection utilities.

### Core Component Annotations
* `@Component`: Generic stereotype for any Spring-managed component.
* `@Service`: Specializes `@Component` for business logic (no added runtime behavior, used for semantic clarity).
* `@Repository`: Specializes `@Component` for data access. Automatically translates database-specific exceptions into Spring’s `DataAccessException` hierarchy.
* `@Controller`: Identifies a class serving web views (returns HTML templates).
* `@RestController`: Combines `@Controller` and `@ResponseBody`. Automatically serializes returned objects directly into JSON/XML payloads.
* `@Bean`: Used inside `@Configuration` classes to register third-party library objects or custom instances in the context.

### Configuration & Properties Annotations
* `@ConfigurationProperties`: Maps external hierarchical configuration properties (e.g. `application.yml`) to a type-safe Java object. Supports validation and relaxed binding.
* `@Value`: Injects a single property value. Avoid for complex nested properties because it lacks structural validation.

---

### Questions & Answers: Dependency Injection & Architecture

#### Q1: What is the difference between `@Bean` and `@Component`? When do you choose one over the other?
**Answer:**
> "`@Component` is a class-level annotation. It uses classpath scanning to automatically detect and register classes as beans. You use it for classes that you write and own.
> `@Bean` is a method-level annotation used inside `@Configuration` classes. You use it to register third-party library classes (e.g., configuring a custom `ObjectMapper` from Jackson or an APISIX gateway client) where you cannot add `@Component` to the class code directly."

#### Q2: How does Spring handle circular dependencies when using Constructor Injection? How do you resolve it?
**Answer:**
> "Spring cannot resolve circular dependencies with Constructor Injection because it requires the dependent beans to be constructed first. When Class A needs Class B, and Class B needs Class A, Spring throws a `BeanCurrentlyInCreationException`.
> **Resolution Strategies:**
> 1. **Refactor (Best Practice):** The presence of a circular dependency indicates poor design. Break the cycle by introducing a third interface or helper class that encapsulates the shared functionality.
> 2. **Lazy Initialization:** Annotate one of the constructor parameters with `@Lazy`. Spring will inject a dynamic proxy instead of the fully instantiated object, deferring resolution until the bean method is actually called."

---

## 3. Exception Handling

### Strategies
* **`@ControllerAdvice` & `@ExceptionHandler`:** A global exception interceptor layer. It intercepts exceptions thrown by controllers and maps them to standard JSON payloads and clean HTTP status codes.
* **`ResponseStatusException` (Spring 5+):** Lightweight alternative to custom exceptions. Allows developers to raise inline exceptions with specific HTTP statuses and reason messages.

---

### Questions & Answers: Exception Handling

#### Q1: Write a global Exception Handler that catches validation errors (`MethodArgumentNotValidException`) and custom business errors.
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidationExceptions(MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error -> 
            errors.put(error.getField(), error.getDefaultMessage())
        );
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(errors);
    }

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(BusinessException ex) {
        ErrorResponse response = new ErrorResponse(ex.getErrorCode(), ex.getMessage(), LocalDateTime.now());
        return ResponseEntity.status(ex.getHttpStatus()).body(response);
    }
}
```

---

## 4. Spring Security Architecture

### Core Architecture Components

```
[ Client Request ] ──► [ FilterChainProxy ]
                             │
                             ├─► [ SecurityContextPersistenceFilter ]
                             ├─► [ UsernamePasswordAuthenticationFilter ]
                             └─► [ FilterSecurityInterceptor ] ──► [ Controller ]
```

* **Security Filter Chain:** Spring Security is built on servlet filters. The `DelegatingFilterProxy` delegates filter requests to Spring-managed `FilterChainProxy` beans.
* **AuthenticationManager:** Interface that verifies credentials. It delegates authentication checks to registered `AuthenticationProvider` implementations.
* **SecurityContextHolder:** Stores the details of the currently authenticated principal (`SecurityContext`) in a `ThreadLocal` storage layout.

### JWT & Stateless Authentication Workflow
1. Client logs in via `/api/login`.
2. Server validates credentials and returns a signed JWT.
3. For subsequent requests, client includes the token in the `Authorization: Bearer <token>` header.
4. A custom filter (e.g. `JwtAuthenticationFilter`) intercepting the request extracts the token, validates the signature, extracts the user details, and registers the user in the `SecurityContextHolder`:
   ```java
   UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
   SecurityContextHolder.getContext().setAuthentication(authentication);
   ```

---

### Questions & Answers: Spring Security

#### Q1: What is the risk of using `SecurityContextHolder.getContext().getAuthentication()` inside asynchronous threads (e.g. `@Async` methods)? How do you fix it?
**Answer:**
> "By default, `SecurityContextHolder` uses a `ThreadLocal` strategy. When you spawn an asynchronous thread using `@Async` or manually submit tasks to an executor pool, the authentication context does not propagate to the new thread, causing requests to run as unauthenticated.
> **Fix:** Configure Spring Security to use the **InheritableThreadLocal** strategy by setting the system property at startup:
> `-Dspring.security.strategy=MODE_INHERITABLETHREADLOCAL`
> Alternatively, wrap execution runners in a `DelegatingSecurityContextExecutor` to automatically propagate context to pooled worker threads."

---

## 5. JPA, Hibernate, & HikariCP Optimization

### Entity Lifecycle States
* **Transient:** Object instantiated but not associated with a Hibernate Session, and has no database identity (no primary key).
* **Managed (Persistent):** Object is associated with an active Hibernate session. Any modifications to its fields are tracked by **Dirty Checking** and flushed to the database.
* **Detached:** Session is closed or cleared. The object has database identity but changes are no longer tracked.
* **Removed:** Scheduled for deletion in the database at the end of the transaction.

### Hibernate Performance Bottlenecks & Fixes
* **N+1 Query Problem:** Occurs when loading a parent entity and accessing its lazy-loaded collection, triggering an additional query for each child row.
  * *Fix:* Use JOIN FETCH queries (`SELECT p FROM Parent p JOIN FETCH p.children`) or `@EntityGraph`.
* **First-Level Cache (L1):** Bound to the Hibernate `Session` (Transaction scope). Prevents duplicate queries within the same transaction.
* **Second-Level Cache (L2):** JVM-wide or distributed cache (using Ehcache or Redis) for read-heavy, write-rare data.

### HikariCP (Connection Pool) Tuning
* Under sizing causes requests to block waiting for connections. Over sizing causes database context switching overhead.
* **Formula:** 
  $$\text{Connections} = ((\text{Core Count} \times 2) + \text{Effective Spindle Count})$$

---

### Questions & Answers: JPA & Hibernate

#### Q1: Explain how `@Transactional` works under the hood in Spring. What are the common causes for transaction rollback failures?
**Answer:**
> "Spring's `@Transactional` uses **Spring AOP proxies**. When a method annotated with `@Transactional` is called:
> 1. The proxy interceptor intercepts the call and obtains a connection from the transaction manager.
> 2. It disables auto-commit on the connection and starts a database transaction.
> 3. It invokes the target method.
> 4. If the method completes successfully, the proxy commits the transaction. If a runtime exception is thrown, it rolls it back.
> **Rollback Failure Causes:**
> 1. **Self-Invocation:** Calling `@Transactional` from another method within the same class bypasses the AOP proxy, meaning no transaction is started.
> 2. **Checked Exceptions:** By default, Spring only rolls back transactions for Runtime/Unchecked exceptions (`RuntimeException` and `Error`). If a Checked exception is thrown (e.g. `IOException`), it will NOT trigger a rollback unless explicitly configured: `@Transactional(rollbackFor = Exception.class)`."

---

## 6. Circuit Breaker & Resilience4j

### Circuit Breaker States

```
            ┌───► [ Closed ] ───┐ (Failure rate > threshold)
            │      ▲            │
  (Success  │      │            ▼
   rate     │  [ Half-Open ] ◄──┘ (Sleep duration window expires)
  satisfied)│      │
            └──────┴─── [ Open ]
```

* **Closed:** Normal operations. Traffic flows through.
* **Open:** Failure threshold reached. Incoming requests are blocked and immediately fail or fallback.
* **Half-Open:** Trial period. A limited volume of traffic is allowed through to test if the downstream service has recovered.

---

### Questions & Answers: Resilience

#### Q1: Write a Spring Boot configuration using Resilience4j to protect an external API call with a fallback method.
```java
@Service
public class OrderProcessingService {

    @CircuitBreaker(name = "paymentService", fallbackMethod = "fallbackPayment")
    public PaymentResponse processPayment(PaymentRequest request) {
        // External call to bKash gateway
        return restTemplate.postForObject("/payment", request, PaymentResponse.class);
    }

    public PaymentResponse fallbackPayment(PaymentRequest request, Throwable ex) {
        // Fallback action in case of gateway failure
        return new PaymentResponse("FAILED", "Alternative fallback payment processing offline");
    }
}
```

---

## 7. Spring Cloud Architecture

### Core Components
* **Spring Cloud Config:** Centralized configuration management using Git or vault repositories, permitting configuration updates without restarting services (using `@RefreshScope`).
* **Eureka (Service Discovery):** Microservices register their locations. Gateway query Eureka to route traffic dynamically.
* **Spring Cloud Gateway:** Reactive API Gateway built on Netty, handling routing, rate limiting, and request modifications.
* **Feign Client:** Declarative REST clients making inter-service communication simple.

---

## 8. Spring Boot Actuator

### Production Monitoring
* `/actuator/health`: Provides application health status, database connectivity checks, and custom health indicators.
* `/actuator/metrics`: Exposes internal application counters (JVM memory, active threads, HTTP requests).
* **Prometheus Integration:** Exposing `/actuator/prometheus` allows Prometheus scraper daemons to collect metrics for visualization inside Grafana dashboards.

---
Spring Boot Study Guide
