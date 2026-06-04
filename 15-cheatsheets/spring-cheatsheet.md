# Spring Boot & Framework Cheatsheet

> Quick-reference guide for Spring Core, Boot configurations, security integration, database transaction scopes, and performance settings

---

## 1. Spring Transaction Management (`@Transactional`)

- **Propagation Settings**:
  - `REQUIRED` (Default): Uses existing transaction; creates one if none exists.
  - `REQUIRES_NEW`: Suspends existing transaction; creates a new independent transaction.
  - `MANDATORY`: Throws an exception if no transaction exists.
- **Isolation Levels**:
  - `READ_COMMITTED` (Default in most DBs): Prevents dirty reads.
  - `SERIALIZABLE`: Highest isolation, slowest performance.
- **Rollback Rules**:
  - By default, Spring only rolls back on `RuntimeException` and `Error`.
  - To force rollback on checked exceptions: `@Transactional(rollbackFor = Exception.class)`.

---

## 2. Spring Performance & In-Memory Optimizations

- **HikariCP Connection Pool**:
  - Maximum Pool Size: `spring.datasource.hikari.maximum-pool-size=20`
  - Connection Timeout: `spring.datasource.hikari.connection-timeout=30000`
- **Asynchronous Execution (`@Async`)**:
  - Must enable with `@EnableAsync` on a configuration class.
  - Set up a custom executor bean to prevent threads from growing infinitely:
```java
@Configuration
public class AsyncConfig {
    @Bean
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("SpringAsync-");
        executor.initialize();
        return executor;
    }
}
```

---

## 3. Spring Boot Configuration Cheat List

| Property | Description | Default | Recommended |
|---|---|---|---|
| `server.tomcat.threads.max` | Max Tomcat execution threads | 200 | 500 (For high traffic) |
| `spring.cache.type` | Target caching provider | None | Redis |
| `management.endpoints.web.exposure.include` | Exposed actuator routes | info,health | health,prometheus |
