# Java Architecture Cheatsheet

> Quick-reference guide for Java 17/21 JVM performance tuning, concurrency APIs, garbage collectors, and memory management

---

## 1. Concurrency & Synchronization APIs

- **`StampedLock`**: Optimistic reads, pessimistic reads/writes.
- **`LongAdder` / `DoubleAdder`**: Striped counter updates for high-contention scenarios. Prefer over `AtomicLong` when write QPS is high and read QPS is moderate.
- **Virtual Threads (Project Loom - Java 21)**: Lightweight user-mode threads managed by the JVM instead of the OS. Created via `Thread.ofVirtual().start(runnable)`. Ideal for I/O-bound scaling.

---

## 2. JVM Garbage Collector Tuning

| GC Type | Focus | Target Use Case | Key Flags |
|---|---|---|---|
| **G1GC** | Balanced | General multi-gigabyte heaps | `-XX:+UseG1GC` `-XX:MaxGCPauseMillis=200` |
| **ZGC** | Ultra-Low Latency | Huge heaps (16GB-16TB), < 1ms pauses | `-XX:+UseZGC` `-XX:+ZGenerational` |
| **Serial GC** | Low Overhead | Small CLI apps, single-CPU containers | `-XX:+UseSerialGC` |

---

## 3. Core Memory Management

```
┌────────────────────────────────────────────────────────┐
│ JVM Memory (JVM Heap + Off-Heap)                       │
│  ┌───────────────────────┐  ┌───────────────────────┐  │
│  │ JVM Heap              │  │ Off-Heap (Metaspace)  │  │
│  │ - Young Gen (Eden/S0) │  │ - Class Metadata      │  │
│  │ - Old Generation      │  │ - JNI Direct Buffers  │  │
│  └───────────────────────┘  └───────────────────────┘  │
└────────────────────────────────────────────────────────┘
```
- **Eden Space**: New objects are allocated here first.
- **Survivor Spaces (S0 / S1)**: Objects that survive minor GC collections copy between survivor spaces.
- **Tenured/Old Generation**: Long-lived objects promoted after surviving age thresholds (e.g. `-XX:MaxTenuringThreshold=15`).
- **OutOfMemoryError (OOM) Analysis**:
  - Dump heap on crash: `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/logs/`
  - Analyze dump files with Eclipse Memory Analyzer (MAT).
