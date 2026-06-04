# Principal Engineer Mock Interviews

> Mock interview transcripts and scenarios focusing on technical execution, code performance, system integration, and JVM/concurrency deep dives
> **Mocks Included**: 1. High-Performance Java Concurrency Debugging · 2. Scaling Database Ingestion

---

## Mock 1: High-Performance Java Concurrency & Lock Contention

### Interview Setting
- **Interviewer**: Principal Engineer / Technical Director
- **Candidate**: Principal Engineer candidate
- **Goal**: Evaluate candidate's understanding of JVM memory management, low-level concurrency APIs (`ReentrantLock`, `StampedLock`, `CAS`), thread pools, and CPU caches.

### Transcript Excerpt

**Interviewer**: We run a transaction collection class that updates merchant balances. Under low load, the latency is sub-millisecond. However, under high concurrency (e.g. 50 threads modifying the same account balance), latency spikes, and CPU utilization hits 100%. We profile the JVM and see massive thread blocking states on `synchronized` blocks. How do you fix this?

**Candidate**:
- **Diagnosis**: The `synchronized` keyword causes threads to block and context switch. When 50 threads try to acquire the same object monitor, the operating system suspends blocked threads and wakes them up later. This context switching causes massive CPU overhead, leading to latency spikes.
- **First Step Optimization**: I will replace the synchronized block with a lock-free atomic model using **Compare-And-Swap (CAS)** operations, such as `AtomicLong` or `LongAdder`.
- **Atomic updates**: If we only update the balance numeric value, `LongAdder` is optimal. It maintains a striped array of variables to distribute write contention across CPU cores, reducing cache-line invalidation traffic.

```java
// Contended synchronized implementation (Before)
public class AccountSynchronized {
    private long balance;
    public synchronized void deposit(long amount) {
        this.balance += amount;
    }
}

// Lock-Free striping implementation (After)
import java.util.concurrent.atomic.LongAdder;
public class AccountAtomic {
    private final LongAdder balance = new LongAdder();
    public void deposit(long amount) {
        this.balance.add(amount);
    }
    public long getBalance() {
        return this.balance.sum();
    }
}
```

**Interviewer**: That works if we only update the balance. What if the transaction logic is complex (e.g. checking status flags and performing conditional deductions) and requires transactional blocks?

**Candidate**:
- If complex updates are needed, I will replace `synchronized` with `StampedLock` (introduced in Java 8).
- **Why StampedLock?**: It supports **Optimistic Reads**. A thread can read values without acquiring a read lock. It returns a stamp value, verifies the stamp afterwards to ensure no write occurred mid-read, and commits. If a write occurred, it falls back to acquire a standard read/write lock. This prevents read operations from blocking each other.

```java
public class AccountStamped {
    private final StampedLock lock = new StampedLock();
    private long balance;

    public void withdrawOptimistic(long amount) {
        long stamp = lock.tryOptimisticRead();
        long currentBalance = this.balance;
        
        // Validate if a write lock was acquired by another thread
        if (!lock.validate(stamp)) {
            // Fall back to a standard read lock
            stamp = lock.readLock();
            try {
                currentBalance = this.balance;
            } finally {
                lock.unlockRead(stamp);
            }
        }
        
        if (currentBalance >= amount) {
            long writeStamp = lock.writeLock();
            try {
                this.balance -= amount;
            } finally {
                lock.unlockWrite(writeStamp);
            }
        }
    }
}
```

---

## Evaluation Rubric

- **Concurrency Knowledge (Critical)**: Candidates must understand JVM lock escalation, CPU context switching, and the difference between pessimistic blocking and optimistic CAS patterns.
- **Java API Familiarity**: Looking for knowledge of `java.util.concurrent` components (`StampedLock`, `LongAdder`, `AtomicFieldUpdater`).
- **Hardware Awareness**: Understanding of CPU cache lines, L1/L2 cache locality, and false sharing issues is a major differentiator.
