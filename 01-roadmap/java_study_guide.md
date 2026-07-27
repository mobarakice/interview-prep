# Java Senior Developer Interview Preparation Guide

This guide covers core Java architecture, modern language features (Java 8 to 25), memory management, concurrency, and OOP/Design Patterns, tailored for senior roles.

---

## 1. Core Concepts, OOP, & Design Patterns

### Core OOP Concepts
* **Polymorphism:** 
  * *Static (Compile-time):* Method overloading. Resolved at compile-time based on method signatures.
  * *Dynamic (Runtime):* Method overriding. Resolved at runtime using **Dynamic Method Dispatch** via the virtual method table (vtable).
* **Inheritance vs. Composition:** Favor composition over inheritance (ISA vs. HASA relationship) to reduce tight coupling.
* **Encapsulation:** Hiding internal state via private fields and exposing controlled behavior via public methods.
* **Abstraction:** Hiding implementation details using interfaces and abstract classes.

### SOLID Principles
* **S**ingle Responsibility (SRP): A class should have only one reason to change.
* **O**pen/Closed (OCP): Classes should be open for extension but closed for modification.
* **L**iskov Substitution (LSP): Subtypes must be substitutable for their base types without altering correctness.
* **I**nterface Segregation (ISP): Clients should not be forced to depend on interfaces they do not use.
* **D**ependency Inversion (DIP): Depend on abstractions, not concretions.

### High-Frequency Design Patterns
* **Creational:** 
  * *Singleton:* Double-checked locking with `volatile` to prevent CPU instruction reordering.
  * *Builder:* Solves telescoping constructors; enforces immutability.
* **Structural:** 
  * *Proxy:* Controls access to an object (e.g. Hibernate lazy loading proxy, Spring AOP transaction proxy).
  * *Decorator:* Dynamically adds responsibilities (e.g. Java I/O streams like `BufferedReader(new FileReader())`).
* **Behavioral:**
  * *Strategy:* Defines a family of algorithms and makes them interchangeable (e.g. custom sorting or payment strategies).
  * *Observer:* One-to-many dependency notification (e.g. Spring Event Listeners).

---

### Questions & Answers: Core OOP & Design Patterns

#### Q1: Write a thread-safe, lazy-initialized Singleton implementation in Java. Explain why the `volatile` keyword is mandatory.
```java
public class ThreadSafeSingleton {
    private static volatile ThreadSafeSingleton instance;

    private ThreadSafeSingleton() {
        // Prevent reflection instantiation
        if (instance != null) {
            throw new IllegalStateException("Already initialized");
        }
    }

    public static ThreadSafeSingleton getInstance() {
        if (instance == null) { // First check (no locking overhead)
            synchronized (ThreadSafeSingleton.class) {
                if (instance == null) { // Second check (under lock)
                    instance = new ThreadSafeSingleton();
                }
            }
        }
        return instance;
    }
}
```
**Answer:**
> "The `volatile` keyword is mandatory to prevent **instruction reordering** by the compiler or CPU. 
> The operation `instance = new ThreadSafeSingleton()` is not atomic; it performs three steps in byte-code:
> 1. Allocate memory.
> 2. Invoke the constructor to initialize fields.
> 3. Assign the memory address to the `instance` variable.
> Without `volatile`, the CPU can reorder steps 2 and 3. If another thread executes `getInstance()` after step 3 but before step 2, it will read a non-null reference to uninitialized memory, leading to a crash or undefined behavior."

#### Q2: How does Spring AOP implement the Proxy Pattern? Compare JDK Dynamic Proxies and CGLIB.
**Answer:**
> "Spring AOP uses the Proxy Pattern to weave cross-cutting concerns (like transactions or security check aspects) around target beans.
> 1. **JDK Dynamic Proxies:** Used if the target class implements at least one interface. It generates a proxy class at runtime that implements the target's interfaces and routes calls through an `InvocationHandler`.
> 2. **CGLIB Proxies:** Used if the target class does *not* implement interfaces. CGLIB dynamically generates a subclass of the target at runtime using bytecode generation (ASM library). Because it subclasses the target, it cannot intercept `final` classes or `final` methods."

---

## 2. Collection Framework, Stream API, & Lambda

### Collection Hierarchy Overview

```
              Iterable
                 │
             Collection
        ┌────────┼────────┐
      List      Set     Queue
```
* **Maps** (e.g., `HashMap`, `TreeMap`, `ConcurrentHashMap`) are part of the Java Collections Utility framework but do not inherit from `Collection`.

### HashMap Internal Mechanics
1. **Hashing:** Calculates hash code of key and maps it to a bucket index using `(n - 1) & hash`.
2. **Collision Resolution:** Collision items are stored in a linked list at the bucket.
3. **Java 8 Treeification:** If a bucket's linked list length exceeds the threshold (`TREEIFY_THRESHOLD = 8`) and the total capacity is at least 64, the linked list is converted into a balanced **Red-Black Tree** to improve lookup latency from $O(n)$ to $O(\log n)$.
4. **Resizing:** Occurs when the size exceeds `capacity * loadFactor` (default `0.75`). Capacity doubles, and elements are rehashed.

### Thread-Safe Collections
* **`Collections.synchronizedMap`:** Wraps a Map, locking the entire collection object for every read/write. Highly bottlenecked under write concurrency.
* **`ConcurrentHashMap` (Java 8+):** No global locking. Uses **CAS (Compare-And-Swap)** instructions for empty buckets and locks only the bucket head node (`synchronized` on the first node) for modifications. Allows lock-free concurrent reads.

### Stream API & Functional Interfaces
* **Streams:** Pipelines that do not store data. They are lazily evaluated: intermediate operations (e.g. `filter`, `map`) return another Stream and do not execute until a terminal operation (e.g. `collect`, `forEach`) is invoked.
* **Core Functional Interfaces:**
  * `Predicate<T>`: `T -> boolean`
  * `Function<T, R>`: `T -> R`
  * `Consumer<T>`: `T -> void`
  * `Supplier<T>`: `void -> T`

---

### Questions & Answers: Collections & Streams

#### Q1: Explain why using a mutable object as a key in a `HashMap` is dangerous.
**Answer:**
> "If you use a mutable object as a key in a `HashMap` and subsequently mutate its fields, the object's `hashCode()` will change. 
> When you try to retrieve the value using that key, the `HashMap` will calculate the new bucket index based on the modified hash code. Because this index is different from the original bucket index where the entry was inserted, the map will fail to find the entry, resulting in a silent data loss or lookup leak, even though the key is physically present inside the map's entries."

#### Q2: How does `Stream.parallelStream()` work under the hood? What are the risks of using it in a web application?
**Answer:**
> "`parallelStream()` splits the stream source using a `Spliterator` and processes chunks concurrently using the common **ForkJoinPool**.
> **Risks in Web Applications:**
> 1. **Thread Starvation:** The common `ForkJoinPool` is shared across the entire JVM. If one thread executes a long-running, blocking I/O task in a parallel stream, it can starve the pool, blocking execution for all other requests.
> 2. **Context Leakage:** ThreadLocals (such as Spring's `@Transactional` context, security contexts, or tenant IDs) do not propagate automatically to worker threads in a parallel stream, leading to authorization or transaction failures."

---

## 3. Java Version Evolution (Java 8 to 25)

### Java 8 & 11 Core Features
* **Java 8:** Lambdas, Streams, `Optional` (prevents NPEs), Default & Static interface methods, New Date/Time API (`java.time`).
* **Java 11:** `var` keyword inside lambda parameters, HTTP Client supporting HTTP/2, launch single-file programs without compilation.

### Java 17 Features
* **Records:** Immutable data classes. Generates constructor, getters, `equals()`, `hashCode()`, and `toString()` automatically.
* **Sealed Classes & Interfaces:** Restricts which classes/interfaces can extend or implement them.
  ```java
  public sealed interface PaymentMethod permits CreditCard, BankTransfer {}
  ```
* **Pattern Matching for Switch (Preview):** Pattern check inside switch statements.
* **Text Blocks:** Multi-line string literals without escape sequences.

### Java 21 Features
* **Virtual Threads (Project Loom):** Lightweight threads that drastically reduce the resource cost of concurrent blocking operations.
* **Record Patterns:** Deconstruct records directly in pattern matching.
  ```java
  if (obj instanceof Point(int x, int y)) { ... }
  ```
* **Sequenced Collections:** Adds `getFirst()`, `getLast()`, `addFirst()`, `addLast()`, `reversed()` to List, Set, Deque interface.

### Java 25 (Planned / Modern Roadmap)
* **Project Valhalla (Value Objects):** Eliminates identity for objects that only store data, allowing the JVM to flatten data structures in memory (similar to primitives), reducing GC and pointer overhead.
* **Flexible Constructor Bodies:** Allows writing code before calling `super(...)` inside constructors.

---

### Questions & Answers: Java Version Evolution

#### Q1: What are Sealed Classes, and how do they benefit compiler safety and Domain-Driven Design (DDD)?
**Answer:**
> "Sealed classes restrict subclassing to a predefined, closed set of permit implementations.
> **Benefits:**
> 1. **Compiler Exhaustiveness Analysis:** When using switch expressions with a sealed hierarchy, the compiler checks that all permitted subclasses are covered. You do not need to add a redundant `default` case.
> 2. **DDD Alignment:** Perfect for modelling algebraic data types (ADTs). You can explicitly represent closed domain concepts (e.g., a `PaymentStatus` that can *only* be `Pending`, `Approved`, or `Failed`)."

#### Q2: Convert a standard Java class into a Java 17 Record and explain how it enforces immutability.
**Answer:**
```java
// Traditional Class
public final class UserDto {
    private final String username;
    private final String email;
    public UserDto(String username, String email) {
        this.username = username;
        this.email = email;
    }
    public String getUsername() { return username; }
    public String getEmail() { return email; }
    // equals, hashCode, toString...
}

// Java 17 Record Equivalent
public record UserDto(String username, String email) {}
```
> "Records enforce immutability because all instance fields are implicitly declared `private final`. No setters are generated. The class is final, preventing inheritance. If a record contains mutable object fields (like a `List`), those objects can still be mutated internally unless wrapped in an unmodifiable collection."

---

## 4. Garbage Collection, Memory Management, & Profiling

### JVM Memory Regions

```
┌────────────────────────────────────────────────────────┐
│ JVM Memory                                             │
├──────────────────────────┬─────────────────────────────┤
│ Heap (Shared)            │ Non-Heap / Metaspace        │
│ [ Young Gen | Old Gen ]  │ [ Class Metadata, JIT ]     │
├──────────────────────────┼─────────────────────────────┤
│ Thread Stacks            │ PC Registers / Native Stacks│
│ [ Local Frames, Refs ]   │                             │
└──────────────────────────┴─────────────────────────────┘
```

* **Heap Memory:** Shared across threads. Stores objects.
  * **Young Generation:** Eden Space, Survivor Space 0 (S0), Survivor Space 1 (S1). Most objects die young (Weak Generational Hypothesis). Minor GC cleans Eden and copies survivors to S0/S1.
  * **Old Generation:** Stores long-lived objects. Promoted from Young Gen after surviving a threshold threshold age (tenuring threshold). Major GC (Full GC) cleans this.
* **Metaspace (Java 8+):** Replaced PermGen. Stores class definitions and metadata in native OS memory (off-heap), preventing out-of-memory errors due to dynamic class loading.

### Garbage Collection Algorithms
* **Parallel GC:** Multi-threaded throughput collector. Pauses application threads during collection (Stop-The-World).
* **G1 GC (Garbage-First):** Incremental collector. Divides heap into equal-sized regions and collects regions with the most garbage first to meet latency goals.
* **ZGC (Zero Garbage Collector - Java 15+):** Ultra-low latency collector. Performs all compaction and collection phases concurrently with application threads, keeping pauses under 1 millisecond even on terabyte-scale heaps.

### Memory Leaks in Java
A memory leak occurs when unused objects are referenced by active GC roots, preventing reclamation.
* *Causes:* Static collections growing without bound, unclosed DB connections/sockets, uncleared `ThreadLocal` variables (especially in application servers like Tomcat), unregistered event listeners.

---

### Questions & Answers: GC & Memory Management

#### Q1: How do you diagnose a memory leak in a production Spring Boot application using profiling tools?
**Answer:**
> "1. **Enable JVM Options:** Run the application with `-XX:+HeapDumpOnOutOfMemoryError` and `-XX:HeapDumpPath` to capture memory state on crashes.
> 2. **Analyze GC Logs:** Use tools like GCEasy to check GC frequency and heap reclamation trends. If heap baseline consumption increases continuously after Full GCs, a leak is present.
> 3. **Capture Heap Dumps:** Use `jmap -dump:format=b,file=heap.hprof <pid>` or `jcmd` in production to capture a snapshot.
> 4. **Use Profilers:** Load the heap dump into **Eclipse MAT (Memory Analyzer Tool)** or **JProfiler**. Use the 'leak suspects' report and analyze the path to GC Roots (excluding weak/soft references) to locate which objects are holding references to dead instances."

#### Q2: What is the risk of using `ThreadLocal` variables in a Spring Boot application running inside a container like Tomcat?
**Answer:**
> "Tomcat uses a thread pool to process incoming HTTP requests. If you set a value in a `ThreadLocal` during a request and fail to call `ThreadLocal.remove()` in a `finally` block or filter interceptor, that data remains bound to the thread.
> When Tomcat reuse that thread to serve a different user request:
> 1. **Data Leak/Security Vulnerability:** The new request can read the previous user's data.
> 2. **Memory Leak:** The object stored in `ThreadLocal` cannot be garbage collected because the thread (GC Root) remains active indefinitely, causing memory consumption to scale until OOM."

---

## 5. Concurrency, Threads, & Virtual Threads

### Thread Life Cycle States

```
   NEW ──► RUNNABLE ──► TERMINATED
            ▲    │
   ┌────────┴────┼────────┐
WAITING  TIMED_WAITING  BLOCKED
```

* **NEW:** Created, `start()` not called.
* **RUNNABLE:** Ready to run or executing.
* **BLOCKED:** Waiting for a monitor lock to enter/re-enter a synchronized block.
* **WAITING:** Waiting indefinitely for another thread (e.g. `object.wait()`, `thread.join()`).
* **TIMED_WAITING:** Waiting for a specified time (e.g. `Thread.sleep(ms)`).
* **TERMINATED:** Finished execution.

### Locks & Synchronizers
* **`synchronized`:** Monitors managed by the JVM. Reentrant, but lacks timeouts or interruptibility.
* **`ReentrantLock`:** Part of `java.util.concurrent.locks`. Adds polling (`tryLock()`), timeouts, and fairness policies.
* **`StampedLock` (Java 8+):** Supports **Optimistic Reading**. You can read data without acquiring a read lock; before committing, validate if a write occurred, falling back to a hard read lock if needed.

### Platform Threads vs. Virtual Threads (Project Loom)

```
+------------------------------------------+------------------------------------------+
| Platform Threads (Traditional)           | Virtual Threads (Java 21)                |
+------------------------------------------+------------------------------------------+
| - 1:1 Mapping to OS kernel threads.      | - M:N mapping to Carrier OS threads.     |
| - High memory footprint (~1MB stack).    | - Ultra-lightweight (~few hundred bytes).|
| - Context switching is expensive.        | - Mounted/demounted from carrier thread  |
| - Thread pools required (limited size).  | - Thread-per-request without pools.      |
+------------------------------------------+------------------------------------------+
```

#### Mounting & Demounting Mechanics
When a Virtual Thread hits a blocking I/O operation (e.g., waiting for database response or socket read), the virtual thread scheduler catches the blocking call, yields the execution context of the virtual thread to the heap, and **demounts** it from its underlying OS Carrier Thread. 
The Carrier Thread is now free to run other virtual threads. Once the I/O event completes, the scheduler mounts the virtual thread back onto an available carrier thread to resume execution.

#### Virtual Thread Pinning
If a virtual thread executes inside a `synchronized` block/method, or calls a Native Method (JNI), it is **pinned** to its OS Carrier Thread. If it performs blocking I/O while pinned, the carrier thread blocks too, neutralizing the performance benefits. 
* *Remedy:* Replace `synchronized` blocks with `ReentrantLock` in virtual thread code.

---

### Questions & Answers: Concurrency & Virtual Threads

#### Q1: Explain why Thread Pools (e.g., `FixedThreadPool`) are obsolete/anti-patterns when using Virtual Threads in Java 21.
**Answer:**
> "Thread pools exist to share expensive resources (OS threads). Platform threads are expensive to create (~1MB memory, kernel context switching).
> Virtual threads are cheap and lightweight. Creating a virtual thread takes bytes and takes microseconds. 
> If you pool virtual threads, you limit concurrency. Instead, allocate a new virtual thread for every single task using `Executors.newVirtualThreadPerTaskExecutor()`. To limit access to a shared resource (like database connections), use a **Semaphore** rather than a thread pool."

#### Q2: What is "Virtual Thread Pinning", and how do you diagnose it in a production application?
**Answer:**
> "Virtual Thread Pinning occurs when a virtual thread cannot yield its execution carrier thread because its stack frame is locked inside a native call or a JVM-monitored `synchronized` block.
> To diagnose pinning:
> 1. Run the JVM with the system property:
>    `-Djdk.tracePinnedThreads=short` or `-Djdk.tracePinnedThreads=full`.
> 2. This prints thread dumps highlighting which class files and code execution paths are pinning the carrier thread.
> 3. Use tools like **JDK Flight Recorder (JFR)** to capture 'Virtual Thread Pinning' events. To resolve it, refactor the code to replace `synchronized` locks with `ReentrantLock` instances."

---

## 6. Functional Programming in Java

### Functional Principles
* **Immutability:** State cannot be modified after construction. Enables thread-safe code without locks.
* **Pure Functions:** Same input always yields the same output, with zero side effects.
* **Referential Transparency:** An expression can be replaced with its value without changing program behavior.
* **Monads:** Encapsulations of computation context. In Java, `Optional` and `Stream` are monads. They allow pipeline transformations (`map`, `flatMap`) without explicit null or loop checks.

---

### Questions & Answers: Functional Programming

#### Q1: What is the difference between `map` and `flatMap` in the Java `Optional` class? Write code to demonstrate.
**Answer:**
> "`map` takes a function that returns a value and wraps it inside an `Optional`.
> `flatMap` takes a function that returns an `Optional` directly, preventing nested optionals (`Optional<Optional<T>>`)."
```java
public class OptionalDemo {
    public record Address(String city) {}
    public record User(String name, Optional<Address> address) {}

    public static void main(String[] args) {
        Optional<User> userOpt = Optional.of(new User("Alice", Optional.of(new Address("Dhaka"))));

        // Using map -> yields Optional<Optional<Address>> (nested)
        Optional<Optional<Address>> nested = userOpt.map(User::address);

        // Using flatMap -> yields Optional<Address> (flattened)
        Optional<Address> address = userOpt.flatMap(User::address);
        
        System.out.println(address.map(Address::city).orElse("Unknown")); // Output: Dhaka
    }
}
```

---
Java Preparation Blueprint
