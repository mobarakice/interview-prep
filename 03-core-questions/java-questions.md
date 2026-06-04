# Java Architect-Level Interview Questions

> 15 questions covering Java 21/25, JVM internals, concurrency, performance, collections, design patterns, and functional programming.
> Cross-references: [Java Cheatsheet](../15-cheatsheets/java-cheatsheet.md) · [Spring Questions](./spring-questions.md)

---

## Q1: Explain Virtual Threads in Java 21 and how they differ from Platform Threads. When would you NOT use them?

### Interviewer's Expectation
Wants to see understanding of Project Loom's motivation, the difference between kernel-managed and JVM-managed threads, and practical judgment about when virtual threads are inappropriate (CPU-bound tasks, synchronized blocks).

### Excellent Answer

Virtual threads are lightweight threads managed by the JVM rather than the operating system. Before Java 21, every `Thread` mapped 1:1 to an OS kernel thread, which are expensive (~1MB stack, costly context switches). Virtual threads are cheap (~few KB), allowing millions to exist concurrently.

```java
// Platform thread - expensive
Thread platformThread = new Thread(() -> doWork());

// Virtual thread - lightweight
Thread virtualThread = Thread.ofVirtual().start(() -> doWork());

// Preferred: ExecutorService with virtual threads
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 100_000).forEach(i ->
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1));
            return i;
        })
    );
}
```

Virtual threads are mounted on carrier threads (platform threads in a ForkJoinPool). When a virtual thread performs a blocking operation (I/O, `Thread.sleep`, `Lock.lock`), it's **unmounted** from the carrier thread, freeing it for other virtual threads. This is the key innovation — blocking becomes cheap.

**Architecture implications**: In a Spring Boot microservice handling 10,000 concurrent requests, traditional thread-per-request models require 10,000 platform threads (~10GB stack memory). With virtual threads, the same workload uses a small pool of carrier threads with negligible memory overhead.

**When NOT to use virtual threads**:
1. **CPU-bound tasks** — Virtual threads don't parallelize CPU work better than platform threads. The carrier pool is bounded by CPU cores.
2. **`synchronized` blocks** — Virtual threads **pin** to the carrier thread inside `synchronized` blocks, negating the benefit. Use `ReentrantLock` instead.
3. **Thread-local heavy code** — Virtual threads create millions of instances, so thread-local storage can cause memory bloat. Use scoped values (`ScopedValue`) instead.
4. **Native code / JNI** — Native calls pin virtual threads to carriers.

### Common Mistakes
- Thinking virtual threads are faster — they're not faster, they're more scalable for I/O-bound workloads
- Not knowing about the pinning problem with `synchronized`
- Confusing virtual threads with reactive programming — virtual threads keep the imperative blocking model
- Assuming virtual threads replace thread pools entirely

### Follow-up Questions
- "How does Spring Boot 3.2+ leverage virtual threads?"
- "Compare virtual threads to the reactive model (WebFlux). When would you choose each?"
- "What is structured concurrency and how does it relate to virtual threads?"

---

## Q2: What are Sealed Classes and Pattern Matching in Java 21? How do they improve domain modeling?

### Interviewer's Expectation
Wants to see understanding of algebraic data types, exhaustive pattern matching, and how these features enable safer domain models with compiler-checked completeness.

### Excellent Answer

Sealed classes restrict which classes can extend a type, creating a closed type hierarchy. Combined with pattern matching in `switch`, they enable exhaustive, compiler-checked handling of all cases — essentially bringing algebraic data types to Java.

```java
// Sealed interface - only these can implement it
public sealed interface PaymentResult
    permits PaymentSuccess, PaymentDeclined, PaymentError {
}

public record PaymentSuccess(String transactionId, BigDecimal amount) implements PaymentResult {}
public record PaymentDeclined(String reason, String declineCode) implements PaymentResult {}
public record PaymentError(Exception cause, boolean retryable) implements PaymentResult {}

// Exhaustive pattern matching with deconstruction
public String handlePayment(PaymentResult result) {
    return switch (result) {
        case PaymentSuccess(var txnId, var amount) ->
            "Payment %s completed: $%s".formatted(txnId, amount);
        case PaymentDeclined(var reason, _) ->
            "Declined: " + reason;
        case PaymentError(var cause, var retryable) when retryable ->
            "Retryable error: " + cause.getMessage();
        case PaymentError(var cause, _) ->
            "Fatal error: " + cause.getMessage();
    };
    // No default needed - compiler verifies exhaustiveness
}
```

**Architectural impact**: In domain-driven design, sealed types replace the fragile "type code + if/else" pattern. When you add a new `PaymentResult` subtype, the compiler forces you to handle it everywhere — eliminating an entire class of runtime bugs. This is transformative for financial systems where missing a case can mean lost money.

In practice, I use sealed interfaces for: command/event hierarchies in CQRS, result types (replacing exceptions for expected failures), and state machines (loan states, order states).

### Common Mistakes
- Not understanding that `permits` clause is required unless subtypes are in the same file
- Forgetting that permitted subtypes must be `final`, `sealed`, or `non-sealed`
- Not leveraging record deconstruction patterns within `switch`
- Using sealed classes when a simple enum would suffice

### Follow-up Questions
- "How would you use sealed classes to model a state machine for loan processing?"
- "What's the difference between sealed classes and the Visitor pattern?"
- "How do guarded patterns (`when` clause) work?"

---

## Q3: Explain the Java Memory Model (JMM), happens-before relationships, and how they affect concurrent programming.

### Interviewer's Expectation
Deep understanding of memory visibility guarantees, happens-before ordering, and the practical implications for writing correct concurrent code. Not just theory — real architectural impact.

### Excellent Answer

The Java Memory Model defines the rules for when writes by one thread become visible to reads by another thread. Without explicit synchronization, the JVM and CPU are free to reorder instructions and cache values — meaning one thread may never see another thread's writes.

**Happens-before relationships** guarantee ordering. If action A happens-before action B, then A's effects are visible to B. Key happens-before rules:

1. **Program order**: Within a single thread, each action happens-before the next
2. **Monitor lock**: Unlock of a monitor happens-before subsequent lock of that monitor
3. **Volatile**: Write to a volatile field happens-before subsequent read of that field
4. **Thread start**: `thread.start()` happens-before any action in the started thread
5. **Thread join**: All actions in a thread happen-before `join()` returns
6. **Transitivity**: If A happens-before B, and B happens-before C, then A happens-before C

```java
// BROKEN: No happens-before between writer and reader
class Broken {
    private boolean ready = false;  // Non-volatile!
    private int value = 0;

    void writer() { value = 42; ready = true; }
    void reader() {
        if (ready) {
            // May print 0! The JIT/CPU can reorder writer's assignments
            // AND reader may see stale cached value
            System.out.println(value);
        }
    }
}

// CORRECT: volatile establishes happens-before
class Correct {
    private volatile boolean ready = false;
    private int value = 0;

    void writer() { value = 42; ready = true; }  // value write happens-before volatile write
    void reader() {
        if (ready) {  // volatile read happens-after volatile write
            System.out.println(value);  // Guaranteed to see 42
        }
    }
}
```

**Architectural implications**: At the architect level, the JMM affects design decisions like:
- **Choosing between `synchronized`, `volatile`, `Atomic*`, and `VarHandle`** — each has different performance/correctness trade-offs
- **Immutable objects are inherently thread-safe** — this is why `record` types and value objects in DDD are powerful in concurrent systems
- **Double-checked locking** must use `volatile` to be correct (many legacy codebases get this wrong)
- **False sharing** in high-performance systems — `@Contended` annotation pads cache lines

### Common Mistakes
- Believing `volatile` provides atomicity (it only provides visibility and ordering)
- Not understanding that `final` fields have special semantics (safely published after constructor completes)
- Thinking `Thread.sleep()` creates a happens-before relationship (it doesn't)
- Confusing JMM guarantees with physical CPU cache coherence protocols

### Follow-up Questions
- "Explain the difference between `volatile`, `Atomic*`, and `VarHandle` in terms of ordering guarantees."
- "What is false sharing and how would you detect it in a high-throughput Java application?"
- "How does the `final` field semantics interact with object publication?"

---

## Q4: Compare GC algorithms in modern JVMs (G1, ZGC, Shenandoah). How do you choose for production systems?

### Interviewer's Expectation
Practical knowledge of GC trade-offs, tuning experience, and the ability to select and configure GC for different workload profiles (latency-sensitive vs throughput-oriented).

### Excellent Answer

| Feature | G1 GC | ZGC | Shenandoah |
|---------|-------|-----|------------|
| **Pause target** | ~200ms (tunable) | < 1ms (sub-millisecond) | < 10ms |
| **Heap size** | Up to TBs | Up to 16 TB | Up to TBs |
| **Concurrent compaction** | Partial (during mixed GCs) | Fully concurrent | Fully concurrent |
| **CPU overhead** | Low-medium | Medium-high | Medium-high |
| **Throughput** | High | Slightly lower | Slightly lower |
| **Default since** | Java 9 | Experimental Java 11, production Java 15 | Java 12 (OpenJDK) |
| **Best for** | General-purpose, balanced | Ultra-low latency | Low latency, broader JDK support |

**G1 GC**: Divides heap into regions, collects "garbage-first" regions (most garbage, least live data). Good default for most applications. Tunable with `-XX:MaxGCPauseMillis=200`.

**ZGC**: Uses colored pointers and load barriers to perform all GC work concurrently. Pause times are sub-millisecond regardless of heap size — pauses only for root scanning (thread stacks, JNI refs). Ideal for latency-sensitive systems like payment gateways.

**Shenandoah**: Similar goals to ZGC but uses Brooks forwarding pointers. Available in more JDK distributions. Good alternative when ZGC isn't available.

**How I choose in production**:

| Workload | Recommended GC | Rationale |
|----------|---------------|-----------|
| Payment processing (P99 < 50ms) | ZGC | Sub-ms pauses prevent tail latency spikes |
| Batch processing / ETL | G1 or Parallel | Throughput matters more than latency |
| Microservice (Spring Boot, moderate load) | G1 | Good balance, well-understood tuning |
| Large heap (>32GB) real-time | ZGC | Pause times don't scale with heap size |
| Kubernetes pods (small heaps <2GB) | Serial or G1 | Simpler, lower overhead for small heaps |

**Tuning approach**: Start with G1 defaults, measure P99 latency, and only switch to ZGC if pause times cause SLA violations. Use GC logs (`-Xlog:gc*`), JFR recordings, and tools like GCViewer/GCEasy to analyze.

### Common Mistakes
- Over-tuning GC without measuring first — "premature GC optimization is the root of all evil"
- Not enabling GC logging in production (`-Xlog:gc*:file=gc.log:time,tags:filecount=5,filesize=50M`)
- Setting heap too large on containers (exceeding cgroup memory limits → OOMKilled)
- Not using `-XX:+UseContainerSupport` in Docker/K8s environments

### Follow-up Questions
- "How do you set JVM heap size in a Kubernetes pod? What's the relationship between `-Xmx` and the container memory limit?"
- "Explain the GC's interaction with NUMA architectures."
- "What are generational ZGC improvements in Java 21?"

---

## Q5: Explain Structured Concurrency in Java 21+. How does it change error handling in concurrent code?

### Interviewer's Expectation
Understanding of the motivation behind structured concurrency (JEP 453), how `StructuredTaskScope` works, and why it's superior to raw `CompletableFuture` for reliability.

### Excellent Answer

Structured concurrency ensures that concurrent subtasks are treated as a unit of work — they start together, complete together, and if one fails, the others are cancelled. It eliminates "thread leaks" and makes concurrent code as readable as sequential code.

```java
// BEFORE: CompletableFuture - error-prone
CompletableFuture<User> userFuture = CompletableFuture.supplyAsync(() -> fetchUser(id));
CompletableFuture<Order> orderFuture = CompletableFuture.supplyAsync(() -> fetchOrder(id));
// If fetchUser throws, fetchOrder keeps running — thread leak!
// If fetchOrder throws, we still wait for fetchUser — wasted work

// AFTER: Structured Concurrency - safe and clear
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Subtask<User> user = scope.fork(() -> fetchUser(id));
    Subtask<Order> order = scope.fork(() -> fetchOrder(id));

    scope.join();           // Wait for all
    scope.throwIfFailed();  // Propagate first failure

    return new UserDashboard(user.get(), order.get());
}
// If any subtask fails, all others are cancelled immediately
// Scope is closed — no thread leaks possible
```

**Key policies**:
- `ShutdownOnFailure` — Cancel all subtasks if any fails (fail-fast)
- `ShutdownOnSuccess` — Cancel remaining once first succeeds (racing/hedged requests)

**Architectural impact**: In microservice architectures, an API often fans out to 3-5 downstream services. Structured concurrency makes this pattern safe by default — no more orphaned HTTP calls consuming resources after the parent request has already failed.

### Common Mistakes
- Confusing structured concurrency with virtual threads — they're complementary but separate features
- Not understanding that `scope.join()` is required before accessing results
- Trying to use structured concurrency for fire-and-forget tasks (wrong tool)
- Not creating custom `StructuredTaskScope` policies for complex orchestration

### Follow-up Questions
- "How would you implement a custom `StructuredTaskScope` that collects partial results?"
- "Compare structured concurrency with reactive programming for service aggregation."
- "How do scoped values (`ScopedValue`) complement structured concurrency?"

---

## Q6: How does ConcurrentHashMap work internally? How is it different from a synchronized HashMap?

### Interviewer's Expectation
Deep understanding of lock striping, CAS operations, the evolution from Java 7 segments to Java 8+ node-level locking, and performance implications at scale.

### Excellent Answer

`ConcurrentHashMap` provides thread-safe access without locking the entire map. Its implementation has evolved significantly:

**Java 7**: Used **segment locking** — the map was divided into 16 segments (by default), each independently lockable. Concurrent writes to different segments didn't block each other. But this limited concurrency to 16 concurrent writers.

**Java 8+**: Redesigned with **node-level locking using CAS + synchronized**:
- Read operations are entirely **lock-free** using `volatile` reads
- Write operations use **CAS** (Compare-And-Swap) for the first node in a bucket
- If the bucket already has nodes, a `synchronized` block locks only that specific bucket's head node
- The tree threshold (TREEIFY_THRESHOLD = 8) converts linked lists to red-black trees for O(log n) lookup in collision-heavy buckets

```java
// Internal structure (simplified, Java 17+)
// Node<K,V>[] table - the backing array

// put() operation:
// 1. hash = spread(key.hashCode())
// 2. index = hash & (table.length - 1)
// 3. If table[index] is null → CAS to insert (no lock!)
// 4. If table[index] is not null → synchronized(table[index]) { ... }
// 5. If chain length > 8 → treeify
```

**Why NOT `Collections.synchronizedMap(new HashMap<>())`?**
- `synchronizedMap` wraps every operation in a single `synchronized(mutex)` block
- Only ONE thread can access the map at a time — reads block reads
- `ConcurrentHashMap` allows fully concurrent reads and highly concurrent writes

**Performance comparison at 64 threads**:
| Operation | ConcurrentHashMap | synchronizedMap |
|-----------|------------------|-----------------|
| Concurrent reads | ~linear scaling | Serialized (1x) |
| Mixed reads/writes (95/5) | Near-linear | Serialized |
| Write-heavy (50/50) | Good (~16-32x) | Serialized |

**Architect considerations**: For distributed caching patterns, `ConcurrentHashMap` is the right choice for local caches, but consider `computeIfAbsent` for cache-loading patterns and be aware that `size()` is an **estimate** (not exact) under concurrency.

### Common Mistakes
- Thinking `ConcurrentHashMap` allows `null` keys or values (it doesn't — unlike `HashMap`)
- Using compound operations without `compute`/`merge` (check-then-act race conditions)
- Not sizing the initial capacity to avoid resizing under concurrent load
- Confusing thread-safety of the map with thread-safety of the values stored in it

### Follow-up Questions
- "How does `computeIfAbsent` prevent the thundering herd problem in cache loading?"
- "What's the difference between `ConcurrentHashMap` and `ConcurrentSkipListMap`?"
- "How would you implement a thread-safe LRU cache in Java?"

---

## Q7: What are Records in Java and when should architects use them in enterprise applications?

### Interviewer's Expectation
Beyond syntax — understanding records as value objects in DDD, their immutability guarantees, and architectural use cases (DTOs, events, API contracts).

### Excellent Answer

Records are immutable data carriers that automatically generate `equals()`, `hashCode()`, `toString()`, constructor, and accessor methods. They are the Java implementation of **value objects**.

```java
// Before records - verbose
public final class Money {
    private final BigDecimal amount;
    private final Currency currency;
    // constructor, getters, equals, hashCode, toString = ~50 lines
}

// With records - concise
public record Money(BigDecimal amount, Currency currency) {
    // Compact constructor for validation
    public Money {
        if (amount.compareTo(BigDecimal.ZERO) < 0)
            throw new IllegalArgumentException("Amount cannot be negative");
        Objects.requireNonNull(currency, "Currency is required");
    }

    // Custom methods
    public Money add(Money other) {
        if (!this.currency.equals(other.currency))
            throw new CurrencyMismatchException(this.currency, other.currency);
        return new Money(this.amount.add(other.amount), this.currency);
    }
}
```

**Architectural use cases**:
1. **DDD Value Objects**: `Money`, `Address`, `DateRange`, `UserId` — immutable, equality by value
2. **DTOs / API responses**: Request/response payloads that cross service boundaries
3. **Event payloads**: CQRS events (`OrderPlaced(orderId, amount, timestamp)`) — immutable by nature
4. **Configuration**: Typed configuration objects bound from Spring `@ConfigurationProperties`
5. **Pattern matching**: Combined with sealed interfaces for exhaustive type handling

**When NOT to use records**: JPA entities (need mutable state, no-arg constructor), classes with complex lifecycle, objects needing inheritance.

### Common Mistakes
- Using records as JPA entities (they lack a no-arg constructor required by JPA)
- Not using compact constructors for validation
- Confusing records with Lombok `@Value` — records are a language feature with better tooling support
- Not leveraging records with sealed interfaces for algebraic data types

### Follow-up Questions
- "Can records implement interfaces? How would you use this in a CQRS event hierarchy?"
- "How do records interact with Jackson serialization/deserialization?"
- "What are the implications of records for API backward compatibility?"

---

## Q8: Explain the Fork/Join Framework and when you'd use it vs. parallel streams vs. virtual threads.

### Interviewer's Expectation
Understanding of work-stealing algorithms, task decomposition, and practical judgment about which concurrency model fits which workload.

### Excellent Answer

The Fork/Join Framework is a work-stealing thread pool designed for **recursive, divide-and-conquer** parallelism. Each thread has a local deque; idle threads **steal** tasks from busy threads' deques.

```java
// Fork/Join for recursive task decomposition
class MergeSort extends RecursiveAction {
    private final int[] array;
    private final int start, end;

    @Override
    protected void compute() {
        if (end - start < THRESHOLD) {
            Arrays.sort(array, start, end); // Base case
            return;
        }
        int mid = (start + end) / 2;
        invokeAll(
            new MergeSort(array, start, mid),
            new MergeSort(array, mid, end)
        );
        merge(array, start, mid, end);
    }
}
```

**Decision matrix**:

| Use Case | Best Tool | Reason |
|----------|-----------|--------|
| CPU-bound recursive decomposition | Fork/Join directly | Work-stealing optimal for unbalanced recursive tasks |
| CPU-bound data parallelism (streams) | Parallel Streams | Simpler API, uses Fork/Join underneath |
| I/O-bound concurrent requests | Virtual Threads | Millions of concurrent I/O without thread exhaustion |
| Fan-out to multiple services | Structured Concurrency | Safe cancellation, error propagation |
| Fire-and-forget async | `CompletableFuture` | Non-blocking composition, callbacks |
| Batch processing with backpressure | Reactive (WebFlux) | Backpressure support, non-blocking I/O |

**Critical pitfall with parallel streams**: They use the **common ForkJoinPool** shared by the entire JVM. One slow parallel stream can starve others. In production, use a custom pool:

```java
ForkJoinPool customPool = new ForkJoinPool(4);
customPool.submit(() ->
    myList.parallelStream()
          .map(this::expensiveOperation)
          .collect(Collectors.toList())
).get();
```

### Common Mistakes
- Using parallel streams for I/O operations (blocks common pool threads)
- Not setting a threshold for Fork/Join — too-fine granularity adds overhead
- Confusing concurrency (virtual threads) with parallelism (Fork/Join)
- Using parallel streams on small collections (overhead > benefit for < ~10K elements)

### Follow-up Questions
- "How does the work-stealing algorithm work in detail?"
- "What happens to the common ForkJoinPool when virtual threads are enabled?"
- "How would you benchmark to decide between parallel streams and sequential processing?"

---

## Q9: What design patterns are most critical at the architect level? How do they differ from typical usage?

### Interviewer's Expectation
Not textbook definitions — wants to see how patterns are applied at system scale, in distributed contexts, and how they influence service architecture.

### Excellent Answer

At the architect level, design patterns operate at multiple scales — object-level, service-level, and system-level:

**1. Strategy Pattern → Plugin Architecture**
At object level, Strategy swaps algorithms. At architect level, it becomes a plugin system:
```java
// Service-level strategy: Payment processor selection
public interface PaymentProcessor {
    PaymentResult process(PaymentRequest request);
}
// Runtime selection based on currency, amount, region
@Component
public class PaymentRouter {
    private final Map<String, PaymentProcessor> processors;
    public PaymentResult route(PaymentRequest req) {
        return processors.get(req.getRegion()).process(req);
    }
}
```

**2. Observer Pattern → Event-Driven Architecture**
Object-level Observer becomes Kafka-based event publishing across services. The pattern scales from in-process to distributed but introduces eventual consistency challenges.

**3. Circuit Breaker → System Resilience**
Not a traditional GoF pattern, but critical at architect level. Resilience4j implements it:
```java
@CircuitBreaker(name = "paymentService", fallbackMethod = "fallback")
@Retry(name = "paymentService")
@Bulkhead(name = "paymentService")
public PaymentResult processPayment(PaymentRequest request) { ... }
```

**4. Decorator → Cross-Cutting Concerns**
At system level, decorators become middleware/filters: logging, authentication, rate limiting, request tracing — each wrapping the core handler.

**5. Builder → Fluent APIs / DSLs**
Complex configuration objects (Kafka producer config, security filter chains) use builder pattern to manage complexity:
```java
SecurityFilterChain filterChain(HttpSecurity http) {
    return http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/public/**").permitAll()
            .anyRequest().authenticated())
        .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
        .build();
}
```

**6. Facade → API Gateway / BFF**
The API Gateway pattern IS the Facade pattern at system scale — providing a simplified interface to a complex microservices backend.

### Common Mistakes
- Listing patterns without explaining system-level application
- Over-applying patterns — "pattern fever" leads to over-engineering
- Not mentioning anti-patterns (God class, distributed monolith, golden hammer)
- Treating patterns as rigid recipes rather than guidelines

### Follow-up Questions
- "How does the Repository pattern interact with CQRS at system scale?"
- "What anti-patterns have you encountered in microservices, and how did you address them?"
- "How do you decide when a pattern introduces more complexity than it solves?"

---

## Q10: How do you approach JVM performance tuning in production? Walk me through your methodology.

### Interviewer's Expectation
Systematic approach to performance analysis — not random flag tweaking. Wants to see profiling methodology, metric-driven decisions, and production-safe practices.

### Excellent Answer

My performance tuning methodology follows a structured pipeline:

**Phase 1: Measure (Don't Guess)**
```bash
# Enable GC logging (always in production)
-Xlog:gc*:file=/logs/gc.log:time,tags:filecount=5,filesize=50M

# Enable JFR continuous recording (low overhead, <2%)
-XX:StartFlightRecording=duration=0,maxsize=500M,dumponexit=true,filename=/logs/jfr.jfr

# Container-aware settings
-XX:+UseContainerSupport
-XX:MaxRAMPercentage=75.0
```

**Phase 2: Identify Bottlenecks**
1. **CPU profiling** with async-profiler (flame graphs) — find hot methods
2. **Allocation profiling** — identify excessive object creation
3. **Lock profiling** — detect contention points
4. **GC analysis** — GCEasy/GCViewer for pause time patterns

**Phase 3: Optimize (In Priority Order)**
1. **Algorithm/data structure fixes** — O(n²) → O(n log n) gives 1000x improvement; no JVM flag does this
2. **Reduce allocations** — Object pooling for hot paths, primitive specialization, avoid autoboxing
3. **GC tuning** — Choose right collector, set heap size, tune generation ratios
4. **Thread pool sizing** — `CPU-bound: cores`, `I/O-bound: cores × (1 + wait/compute)`
5. **JIT warmup** — `-XX:+TieredCompilation` (default), consider CDS/AOT for startup

**Phase 4: Validate**
Run load tests with production-like traffic patterns. Compare P50, P95, P99, P99.9 latencies — not just averages.

```
Before: P99 = 450ms, GC pauses = 200ms max
After:  P99 = 85ms, GC pauses = 2ms max (switched to ZGC)
```

### Common Mistakes
- Tuning JVM flags without profiling first ("flag-driven development")
- Optimizing average latency instead of tail latency (P99, P99.9)
- Setting `-Xms` and `-Xmx` to different values (causes unnecessary GC during heap growth)
- Not accounting for container memory limits (JVM + native + off-heap ≤ container limit)

### Follow-up Questions
- "How do you handle a production OutOfMemoryError? Walk me through your investigation process."
- "What's the difference between heap and off-heap memory? When would you use off-heap?"
- "How would you tune a Spring Boot application for sub-100ms P99 latency?"

---

## Q11: Explain Java's type system evolution — generics, type erasure, and what Valhalla (value types) will change.

### Interviewer's Expectation
Understanding of generics limitations (type erasure), workarounds, and awareness of Project Valhalla's impact on performance and API design.

### Excellent Answer

**Type Erasure**: Java generics are a compile-time feature. At runtime, `List<String>` and `List<Integer>` are both just `List`. This was a backward-compatibility decision (Java 5 had to run on Java 1.4 JVMs).

**Practical implications**:
```java
// Cannot do at runtime:
if (obj instanceof List<String>) { }  // Compile error
new T();                               // Cannot instantiate type parameter
T[] array = new T[10];                 // Cannot create generic array

// Workarounds:
// 1. Type tokens
public <T> T deserialize(String json, Class<T> type) {
    return objectMapper.readValue(json, type);
}

// 2. Super type tokens (Jackson's TypeReference)
List<User> users = objectMapper.readValue(json, new TypeReference<List<User>>() {});
```

**Project Valhalla (value types)**: Will introduce `value class` — objects without identity, stored inline (no heap allocation, no pointer indirection):
```java
// Future Java (Valhalla)
value class Point {
    int x;
    int y;
}
// Point[] is stored as flat array of (int, int) pairs — no object headers, no pointers
// This eliminates billions of unnecessary allocations in numeric/financial code
```

**Impact on architecture**: Value types will make records even more powerful — a `Money` value type stored in an array will have the performance of primitive arrays while maintaining type safety. This fundamentally changes how we design financial computation engines.

### Common Mistakes
- Not understanding why generic arrays are disallowed (`new T[]` vs `new Object[]` covariance issues)
- Confusing reified generics (C#) with Java's erased generics
- Not knowing about type witnesses and intersection types
- Ignoring the performance cost of boxing in generic collections (`List<Integer>` vs `int[]`)

### Follow-up Questions
- "How does type erasure affect serialization frameworks like Jackson?"
- "What are the implications of Valhalla for collections like `List<int>`?"
- "Explain the difference between covariance, contravariance, and invariance with `? extends` and `? super`."

---

## Q12: How do you approach memory leak detection and resolution in Java applications?

### Interviewer's Expectation
Systematic diagnostic approach, tooling knowledge, and real-world experience identifying common leak patterns in Spring Boot/enterprise applications.

### Excellent Answer

**Common leak patterns in enterprise Java**:
1. **Static collections growing unboundedly** — `static Map` used as a cache without eviction
2. **Listener/callback registration without cleanup** — Event listeners in Spring that aren't removed
3. **ClassLoader leaks** — Hot-reloading in dev/staging retaining old classloaders
4. **Connection/resource leaks** — JDBC connections not closed, `InputStream` not closed
5. **ThreadLocal not cleaned** — Especially dangerous with thread pools (threads are reused)

**Diagnostic methodology**:

```bash
# Step 1: Detect (Prometheus/Grafana alerting on heap usage trend)
# Old Gen growing linearly over hours → likely leak

# Step 2: Capture heap dumps
jcmd <pid> GC.heap_dump /tmp/heap.hprof
# Or trigger on OOM:
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/logs/

# Step 3: Analyze with Eclipse MAT or VisualVM
# Look for: Dominator tree, retained size, leak suspects report
# Key question: What is retaining these objects? (GC roots → reference chain)

# Step 4: In production (if can't take heap dump)
jcmd <pid> GC.class_histogram | head -30
# Shows per-class instance count — repeated runs reveal growing classes
```

**Spring Boot specific leaks**:
```java
// LEAK: @EventListener without cleanup
@EventListener
public void onEvent(MyEvent event) {
    cache.put(event.getId(), event);  // Cache grows forever!
}

// FIX: Use bounded cache
private final Cache<String, MyEvent> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(Duration.ofHours(1))
    .build();
```

### Common Mistakes
- Confusing high memory usage with a memory leak (some apps legitimately use lots of memory)
- Only looking at heap — native memory leaks (Direct ByteBuffers, JNI) won't show in heap dumps
- Not using `-XX:NativeMemoryTracking=summary` for native memory analysis
- Fixing symptoms (increasing heap) instead of root cause (finding the leak)

### Follow-up Questions
- "How would you detect a native memory leak vs. a heap memory leak?"
- "What's the difference between a soft leak and a hard leak?"
- "How does Caffeine cache differ from Guava cache, and why would you choose it?"

---

## Q13: What's new in Java 25 and how does it impact enterprise architecture?

### Interviewer's Expectation
Awareness of the latest Java release, ability to assess which features are production-ready, and architectural judgment about adoption timing.

### Excellent Answer

Java 25 (September 2025 LTS) builds on the foundations laid by Java 21 (previous LTS):

**Key Features for Enterprise Architects**:

1. **Stable Structured Concurrency** — Now finalized. Safe fan-out/fan-in patterns become first-class. This changes how we build service aggregation layers.

2. **Scoped Values (finalized)** — Thread-local replacement for virtual threads. Immutable, inherited by child scopes:
```java
private static final ScopedValue<UserContext> CONTEXT = ScopedValue.newInstance();

ScopedValue.where(CONTEXT, userContext).run(() -> {
    // All code in this scope (including virtual thread children) sees the context
    processRequest();
});
```

3. **String Templates (if finalized)** — Type-safe string interpolation with processors:
```java
String query = SQL."SELECT * FROM users WHERE id = \{userId}";
// SQL processor validates and parameterizes — prevents SQL injection by design
```

4. **Primitive Patterns in instanceof/switch** — Can now pattern match on primitives.

5. **Stream Gatherers (finalized)** — Custom intermediate stream operations:
```java
// Window function — group stream into fixed-size windows
stream.gather(Gatherers.windowFixed(3))
      .forEach(window -> process(window));
```

6. **Class-File API** — Standard API for reading/writing bytecode, replacing ASM/Byte Buddy for framework authors.

**Migration Strategy for Enterprise**:
- **Java 17 → 21 → 25**: Skip intermediate non-LTS versions
- **Test virtual threads** with existing blocking code (Spring Boot 3.2+ ready)
- **Adopt records and sealed classes** in new code, don't rewrite existing
- **Enable ZGC generational** by default for latency-sensitive services

### Common Mistakes
- Rushing to adopt preview features in production
- Not testing virtual thread compatibility with existing `synchronized` code
- Assuming all libraries support the latest Java version
- Not planning for the removal of deprecated APIs (Security Manager removed)

### Follow-up Questions
- "What's your strategy for upgrading a large enterprise from Java 17 to Java 25?"
- "How do you evaluate whether to adopt a preview feature?"
- "What testing strategy ensures virtual threads work correctly with your existing codebase?"

---

## Q14: How do you design for testability at the architecture level in Java?

### Interviewer's Expectation
Beyond unit testing — architecture-level testing strategy including contract testing, integration testing, and how architectural choices (Hexagonal, DI) enable testing.

### Excellent Answer

**Architecture-level testability is designed in, not bolted on.** Key principles:

**1. Hexagonal Architecture enables testing at every boundary**:
```
                  ┌──────────────────────┐
   Test Adapters ──┤     Application      ├── Test Adapters
   (Mock HTTP)    │    (Domain Logic)     │  (In-Memory DB)
                  │    Ports & Adapters    │
                  └──────────────────────┘
```

```java
// Port (interface) - testable in isolation
public interface PaymentGateway {
    PaymentResult charge(Money amount, PaymentMethod method);
}

// Test adapter
public class FakePaymentGateway implements PaymentGateway {
    private final Map<String, PaymentResult> programmedResults = new HashMap<>();
    public PaymentResult charge(Money amount, PaymentMethod method) {
        return programmedResults.getOrDefault(method.getId(), PaymentResult.declined("fake"));
    }
}
```

**2. Testing pyramid for microservices**:

| Level | Tool | What it Tests | Speed |
|-------|------|---------------|-------|
| Unit | JUnit 5, Mockito | Domain logic, business rules | ~ms |
| Integration | Testcontainers | DB, Kafka, Redis with real containers | ~seconds |
| Contract | Pact, Spring Cloud Contract | API contracts between services | ~seconds |
| Component | SpringBootTest + WireMock | Single service end-to-end | ~seconds |
| E2E | Playwright, Cypress | Full system through UI | ~minutes |

**3. Testcontainers for real integration tests**:
```java
@Testcontainers
@SpringBootTest
class PaymentServiceIT {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    @Container
    static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));

    @Test
    void shouldProcessPaymentAndPublishEvent() {
        // Test with real PostgreSQL and Kafka — not mocks
    }
}
```

**4. Contract testing prevents integration breakage**:
```java
// Provider side (Spring Cloud Contract)
@SpringBootTest
@AutoConfigureStubRunner
class OrderServiceContractTest {
    // Verifies that OrderService fulfills the contracts defined by its consumers
    // If PaymentService expects a specific response format, this test catches breakage
}
```

### Common Mistakes
- Mocking everything (tests pass but system breaks in production)
- Not using Testcontainers (testing against H2 instead of PostgreSQL → SQL dialect differences)
- Skipping contract tests (integration breaks discovered only in staging/production)
- Writing tests after the architecture is set (testability should drive design)

### Follow-up Questions
- "How do you test Saga patterns across microservices?"
- "What's your approach to testing Kafka consumers and producers?"
- "How do you handle test data management across microservices?"

---

## Q15: Explain functional programming concepts in Java. How do you apply them in enterprise architecture?

### Interviewer's Expectation
Beyond lambda syntax — understanding immutability, pure functions, composition, and how FP principles improve system reliability in enterprise contexts.

### Excellent Answer

**Core FP principles applied to enterprise Java**:

**1. Immutability → Thread Safety + Event Sourcing**
```java
// Immutable event (record)
public record OrderPlaced(UUID orderId, UUID customerId, List<OrderLine> lines, Instant timestamp) {
    public OrderPlaced {
        lines = List.copyOf(lines); // Defensive copy — truly immutable
    }
}
```
In event-sourced systems, every state change is an immutable event. The current state is derived by folding events:
```java
Order currentState = events.stream()
    .reduce(Order.empty(), Order::apply, (a, b) -> b);
```

**2. Pure Functions → Testable Business Logic**
```java
// Pure function — same input always gives same output, no side effects
public record PricingResult(BigDecimal total, BigDecimal discount) {}

public PricingResult calculatePrice(List<OrderLine> lines, DiscountPolicy policy) {
    BigDecimal subtotal = lines.stream()
        .map(line -> line.price().multiply(BigDecimal.valueOf(line.quantity())))
        .reduce(BigDecimal.ZERO, BigDecimal::add);
    BigDecimal discount = policy.apply(subtotal);
    return new PricingResult(subtotal.subtract(discount), discount);
}
// Easily testable — no mocks needed, no database, no Spring context
```

**3. Function Composition → Pipeline Architecture**
```java
// Request processing pipeline using Function composition
Function<Request, Request> authenticate = this::validateToken;
Function<Request, Request> authorize = this::checkPermissions;
Function<Request, Request> validate = this::validatePayload;
Function<Request, Request> enrich = this::addMetadata;

Function<Request, Request> pipeline = authenticate
    .andThen(authorize)
    .andThen(validate)
    .andThen(enrich);

Response response = pipeline.andThen(this::process).apply(request);
```

**4. Optional/Either → Railway-Oriented Programming**
```java
// Replace exceptions with explicit result types for expected failures
public sealed interface Result<T> permits Success, Failure {
    <R> Result<R> flatMap(Function<T, Result<R>> f);
}
public record Success<T>(T value) implements Result<T> { ... }
public record Failure<T>(String error) implements Result<T> { ... }

// Chain operations — first failure short-circuits
Result<Order> result = validateCustomer(customerId)
    .flatMap(customer -> validateInventory(items))
    .flatMap(inventory -> createOrder(customer, inventory))
    .flatMap(order -> processPayment(order));
```

### Common Mistakes
- Overusing streams where a simple for-loop is clearer
- Not understanding lazy evaluation in streams (terminal operations trigger execution)
- Using `Optional` as method parameters or fields (it's designed for return types)
- Forcing FP style in inherently stateful code (database operations, I/O)

### Follow-up Questions
- "How does functional error handling (Either/Result types) compare to exceptions in microservices?"
- "How would you apply the monad pattern in a Spring Boot service?"
- "What's the performance impact of heavy stream usage vs. imperative loops?"
