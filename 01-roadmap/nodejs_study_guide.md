# Node.js Senior Developer Study Guide

This guide covers Node.js event loop mechanics, streams, backpressure management, multi-threading (Worker Threads), V8 memory profiling, and framework configurations (Express/NestJS/Fastify).

---

## 1. Node.js Event Loop Architecture

Node.js executes JavaScript on a single thread. It achieves asynchronous concurrency by delegating I/O operations to the operating system or the **libuv** thread pool.

### Event Loop Phases

```
                  ┌──────────────────────────┐
            ┌────►│          Timers          │ (setTimeout, setInterval)
            │     └────────────┬─────────────┘
            │                  ▼
            │     ┌──────────────────────────┐
            │     │     Pending Callbacks    │ (Deferred I/O errors)
            │     └────────────┬─────────────┘
            │                  ▼
            │     ┌──────────────────────────┐
            │     │       Idle / Prepare     │ (Internal engine operations)
            │     └────────────┬─────────────┘
            │                  ▼
            │     ┌──────────────────────────┐
            │     │           Poll           │ (Executes I/O callbacks)
            │     └────────────┬─────────────┘
            │                  ▼
            │     ┌──────────────────────────┐
            │     │          Check           │ (setImmediate callbacks)
            │     └────────────┬─────────────┘
            │                  ▼
            │     ┌──────────────────────────┐
            └─────┤      Close Callbacks     │ (socket.on('close'))
                  └──────────────────────────┘
```

* **Timers:** Executes callbacks scheduled by `setTimeout()` and `setInterval()`.
* **Pending Callbacks:** Executes I/O callbacks deferred from the previous loop iteration.
* **Poll:** Retrieves and processes new I/O events. If the poll queue is empty, the loop waits for incoming I/O events unless callbacks are scheduled in the Check phase.
* **Check:** Executes callbacks scheduled by `setImmediate()`.
* **Close Callbacks:** Processes connection termination callbacks (e.g. `socket.on('close')`).

### Microtask Queue vs. Macrotask Queue
Microtasks are executed immediately after the current operation finishes, before the event loop moves to the next phase.
* **Microtasks:** `process.nextTick()` (highest execution priority) and resolved Promises.
* **Macrotasks:** `setTimeout`, `setInterval`, `setImmediate`, and I/O callbacks.

---

## 2. Streams & Backpressure Management

Streams allow processing large datasets (like files or database exports) in chunks without loading the entire payload into RAM.

### Stream Types
1. **Readable:** Source of data (e.g. `fs.createReadStream()`).
2. **Writable:** Destination for data (e.g. `fs.createWriteStream()`).
3. **Duplex:** Readable and Writable (e.g. TCP Socket).
4. **Transform:** A Duplex stream that modifies data during read/write (e.g. `zlib.createGzip()`).

### Managing Backpressure
Backpressure occurs when a Readable stream sends data faster than a Writable stream can process it. The Writable stream buffer fills up, causing high memory usage.
* **Mitigation:** Use `stream.pipeline()` to connect streams. It automatically manages backpressure, pausing the reader when write buffers are full, and handles cleanups:
  ```typescript
  import { pipeline } from 'stream/promises';
  import fs from 'fs';
  import zlib from 'zlib';

  await pipeline(
      fs.createReadStream('large_input.txt'),
      zlib.createGzip(),
      fs.createWriteStream('compressed_output.txt.gz')
  );
  ```

---

## 3. Concurrency & Parallelism

### 1. Child Processes
Spawns independent operating system processes using `child_process.fork()`. Communicates via IPC (Inter-Process Communication). Each process has its own memory space and V8 instance.

### 2. Worker Threads (`worker_threads`)
Runs multiple threads inside the same process. Threads share memory space using `SharedArrayBuffer`, making them suitable for CPU-heavy tasks:
```typescript
// main.ts
import { Worker } from 'worker_threads';

public async void runCpuTask() {
    const worker = new Worker('./worker.js');
    worker.postMessage({ data: 'process_telemetry' });
    worker.on('message', result => {
        console.log('Result from worker:', result);
    });
}
```

### 3. Clustering
Spawns multiple instances of the Node.js process (typically matching CPU core counts) sharing the same network port. A master process distributes incoming HTTP requests using a round-robin load balancing strategy.

---

## 4. V8 Memory Management & Leaks

### V8 Garbage Collector
* **Scavenge (Young Generation):** Fast, minor garbage collection for objects with short lifespans. Uses a Copying algorithm.
* **Mark-Sweep-Compact (Old Generation):** Scans the heap, identifies unreachable objects, and deallocates memory. Runs concurrently to minimize latency.

### Common Memory Leaks
* **Unintentional Global Variables:** Variables not declared with `let` or `const` attach to the global object and are never garbage collected.
* **Uncleared Intervals/Timers:** Callbacks inside `setInterval` retain references to variables in their parent scope.
* **ThreadLocals / AsyncLocalStorage:** Retaining references inside context scopes after the request completes.

---

## 5. Web Framework Architectures

* **Express:** Unopinionated, middleware-based framework. Runs synchronously; unhandled promise rejections in async middleware can crash the server unless wrapped in a try-catch.
* **Fastify:** High-performance, schema-first framework. Built-in support for async/await, input validation with Ajv, and faster routing compared to Express.
* **NestJS:** Highly structured, opinionated TypeScript framework. Uses decorators and Dependency Injection (similar to Angular/Spring Boot). Built on top of Express or Fastify.

---

### Questions & Answers: Node.js

#### Q1: Explain the difference between `process.nextTick()` and `setImmediate()`. Which one executes first?
**Answer:**
> "`process.nextTick()` is not part of the event loop phases. It executes immediately after the current operation completes, before the event loop transitions to the next phase.
> `setImmediate()` schedules callbacks to run during the Check phase of the event loop.
> **Execution Order:** `process.nextTick()` always executes before `setImmediate()`. If you call both in the same block of code, the `process.nextTick()` callback runs first."

#### Q2: Write a NestJS Controller that uses Zod to validate request inputs and handles database errors using a filter interceptor.
```typescript
// dto.ts
import { z } from 'zod';

export const CreateTenantSchema = z.object({
  name: z.string().min(3),
  email: z.string().email(),
});

export type CreateTenantDto = z.infer<typeof CreateTenantSchema>;

// controller.ts
import { Controller, Post, Body, UsePipes, UseFilters } from '@nestjs/common';
import { ZodValidationPipe } from './zod-validation.pipe'; // Custom validation pipe
import { DatabaseExceptionFilter } from './db-exception.filter'; // Custom exception filter

@Controller('tenants')
@UseFilters(DatabaseExceptionFilter)
export class TenantController {
  @Post()
  @UsePipes(new ZodValidationPipe(CreateTenantSchema))
  async create(@Body() createTenantDto: CreateTenantDto) {
    return this.tenantService.create(createTenantDto);
  }
}
```

---
Node.js Study Guide
