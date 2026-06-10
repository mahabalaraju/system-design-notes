# Java Concurrency — Complete Notes

> Advanced reference covering Java's concurrency utilities, memory model, thread management, and concurrent collections with production-grade examples.

---

## Table of Contents

- [Java Memory Model & Happens-Before](#java-memory-model--happens-before)
- [volatile](#volatile)
- [synchronized](#synchronized)
- [Locks — java.util.concurrent.locks](#locks--javautilconcurrentlocks)
  - [ReentrantLock](#reentrantlock)
  - [ReadWriteLock](#readwritelock)
  - [StampedLock](#stampedlock)
- [Atomic Classes](#atomic-classes)
- [Concurrent Collections](#concurrent-collections)
  - [ConcurrentHashMap](#concurrenthashmap)
  - [CopyOnWriteArrayList](#copyonwritearraylist)
  - [CopyOnWriteArraySet](#copyonwritearrayset)
  - [ConcurrentLinkedQueue & Deque](#concurrentlinkedqueue--deque)
  - [BlockingQueue Implementations](#blockingqueue-implementations)
- [Synchronisers](#synchronisers)
  - [CountDownLatch](#countdownlatch)
  - [CyclicBarrier](#cyclicbarrier)
  - [Semaphore](#semaphore)
  - [Phaser](#phaser)
  - [Exchanger](#exchanger)
- [Executor Framework & Thread Pools](#executor-framework--thread-pools)
  - [ThreadPoolExecutor](#threadpoolexecutor)
  - [Built-in Pool Factories](#built-in-pool-factories)
  - [ForkJoinPool](#forkjoinpool)
  - [CompletableFuture](#completablefuture)
- [Thread Lifecycle & Interruption](#thread-lifecycle--interruption)
- [Common Concurrency Problems](#common-concurrency-problems)
- [Best Practices & Decision Guide](#best-practices--decision-guide)
- [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Java Memory Model & Happens-Before

The Java Memory Model (JMM) defines **when writes by one thread are guaranteed to be visible to reads by another**.

Without visibility guarantees, the JVM and CPU are free to:
- Reorder instructions
- Cache values in registers or CPU caches
- Never flush a write to main memory

### Happens-Before Rules

A happens-before (HB) relationship guarantees that all writes **before** the HB edge are visible to all reads **after** it.

| Rule | HB edge created |
|---|---|
| Program order | Statement A before statement B in same thread |
| Monitor lock | `unlock()` HB → `lock()` on same monitor |
| Volatile write | `write(v)` HB → `read(v)` of same volatile variable |
| Thread start | `thread.start()` HB → first action in that thread |
| Thread join | Last action in thread HB → `thread.join()` returns |
| Static initializer | Static init HB → any access to that class |
| `final` field | Write in constructor HB → any read after constructor returns |

### Visibility Example

```java
// Without synchronisation — y may never see x = 1
int x = 0;
boolean ready = false;

// Thread 1
x = 1;
ready = true; // reordering: ready may be set before x

// Thread 2
while (!ready) {}
System.out.println(x); // may print 0!
```

---

## volatile

`volatile` provides two guarantees:
1. **Visibility** — writes are immediately flushed to main memory; reads always from main memory
2. **Ordering** — prevents reordering across the volatile access (full memory barrier)

`volatile` does **not** provide atomicity for compound actions (check-then-act, read-modify-write).

```java
// ✅ Correct use — simple flag
private volatile boolean running = true;

public void stop() { running = false; }

public void run() {
    while (running) { /* work */ }
}
```

```java
// ❌ Not atomic — volatile doesn't help here
private volatile int counter = 0;
counter++; // read-modify-write: NOT atomic, use AtomicInteger
```

### Double-Checked Locking (requires volatile)

```java
public class Singleton {
    private static volatile Singleton instance; // volatile is mandatory

    public static Singleton getInstance() {
        if (instance == null) {                  // first check (no lock)
            synchronized (Singleton.class) {
                if (instance == null) {          // second check (with lock)
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

Without `volatile`, the JVM may publish a partially constructed object — the reference assignment can be reordered before the constructor completes.

---

## synchronized

`synchronized` provides **mutual exclusion + visibility** (both atomicity and happens-before).

```java
// Method-level lock — locks on `this`
public synchronized void increment() { count++; }

// Block-level — finer granularity
public void increment() {
    synchronized (this) { count++; }
}

// Static method — locks on Class object
public static synchronized void staticMethod() { /* ... */ }

// Private lock object — best practice (prevents external lock contention)
private final Object lock = new Object();
public void safeMethod() {
    synchronized (lock) { /* ... */ }
}
```

### Intrinsic Lock Properties
- **Reentrant** — same thread can re-acquire without deadlock
- **Non-interruptible** — cannot be interrupted while waiting
- **No timeout** — waits indefinitely
- **Not fair** — no ordering guarantee among waiting threads

---

## Locks — java.util.concurrent.locks

### ReentrantLock

More flexible than `synchronized` — supports tryLock, timeout, and interruptible waiting.

```java
private final ReentrantLock lock = new ReentrantLock(true); // true = fair

public void transfer(Account from, Account to, BigDecimal amount) {
    lock.lock();
    try {
        from.debit(amount);
        to.credit(amount);
    } finally {
        lock.unlock(); // always in finally
    }
}

// Try with timeout — avoids indefinite blocking
public boolean tryTransfer(Account from, Account to, BigDecimal amount)
        throws InterruptedException {
    if (lock.tryLock(500, TimeUnit.MILLISECONDS)) {
        try {
            from.debit(amount);
            to.credit(amount);
            return true;
        } finally {
            lock.unlock();
        }
    }
    return false; // couldn't acquire in time
}

// Interruptible lock — allows thread interruption while waiting
public void interruptibleOp() throws InterruptedException {
    lock.lockInterruptibly();
    try { /* ... */ }
    finally { lock.unlock(); }
}
```

**Condition variables** (replaces `wait()/notify()`):

```java
private final ReentrantLock lock = new ReentrantLock();
private final Condition notFull  = lock.newCondition();
private final Condition notEmpty = lock.newCondition();

public void put(T item) throws InterruptedException {
    lock.lock();
    try {
        while (isFull()) notFull.await();
        add(item);
        notEmpty.signal();
    } finally {
        lock.unlock();
    }
}

public T take() throws InterruptedException {
    lock.lock();
    try {
        while (isEmpty()) notEmpty.await();
        T item = remove();
        notFull.signal();
        return item;
    } finally {
        lock.unlock();
    }
}
```

---

### ReadWriteLock

Allows **multiple concurrent readers** or **one exclusive writer** — not both.

```java
private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
private final Lock readLock  = rwLock.readLock();
private final Lock writeLock = rwLock.writeLock();
private final Map<String, String> cache = new HashMap<>();

public String get(String key) {
    readLock.lock();
    try {
        return cache.get(key); // concurrent reads allowed
    } finally {
        readLock.unlock();
    }
}

public void put(String key, String value) {
    writeLock.lock();
    try {
        cache.put(key, value); // exclusive write
    } finally {
        writeLock.unlock();
    }
}
```

**When to use:** Read-heavy workloads where reads far outnumber writes.
**Pitfall:** Write starvation can occur under very high read load (use fair mode or StampedLock).

---

### StampedLock

Java 8+. Adds **optimistic reads** — no lock acquired, just a stamp; validate before using the data.

```java
private final StampedLock sl = new StampedLock();
private double x, y;

// Optimistic read — fastest path, no lock acquired
public double distanceFromOrigin() {
    long stamp = sl.tryOptimisticRead();
    double cx = x, cy = y;
    if (!sl.validate(stamp)) {     // check if a write happened
        stamp = sl.readLock();     // fall back to real read lock
        try { cx = x; cy = y; }
        finally { sl.unlockRead(stamp); }
    }
    return Math.sqrt(cx * cx + cy * cy);
}

// Write
public void move(double deltaX, double deltaY) {
    long stamp = sl.writeLock();
    try { x += deltaX; y += deltaY; }
    finally { sl.unlockWrite(stamp); }
}

// Lock upgrade — read → write
public void conditionalWrite(double threshold) {
    long stamp = sl.readLock();
    try {
        while (x < threshold) {
            long ws = sl.tryConvertToWriteLock(stamp);
            if (ws != 0L) { stamp = ws; x = threshold; break; }
            else { sl.unlockRead(stamp); stamp = sl.writeLock(); }
        }
    } finally {
        sl.unlock(stamp);
    }
}
```

**StampedLock vs ReadWriteLock:**
- StampedLock is faster due to optimistic reads
- StampedLock is **not reentrant** — same thread cannot re-acquire
- StampedLock has no Condition support

---

## Atomic Classes

Lock-free thread safety via **Compare-And-Swap (CAS)** — a single CPU instruction.

```
CAS(location, expected, newValue):
  if (*location == expected) { *location = newValue; return true; }
  else { return false; } // retry
```

### Core Atomic Types

```java
AtomicInteger    counter = new AtomicInteger(0);
AtomicLong       longVal = new AtomicLong(0L);
AtomicBoolean    flag    = new AtomicBoolean(false);
AtomicReference<User> ref = new AtomicReference<>(new User());

// Common operations
counter.incrementAndGet();          // ++counter
counter.getAndIncrement();          // counter++
counter.addAndGet(5);               // counter += 5
counter.compareAndSet(10, 20);      // CAS: if 10, set 20
counter.updateAndGet(x -> x * 2);  // lambda update
counter.accumulateAndGet(5, Integer::sum); // accumulate
```

### AtomicReference — lock-free object swaps

```java
AtomicReference<BigDecimal> price = new AtomicReference<>(BigDecimal.ZERO);

// Lock-free update
BigDecimal prev, next;
do {
    prev = price.get();
    next = prev.add(delta);
} while (!price.compareAndSet(prev, next));
```

### LongAdder & LongAccumulator (Java 8+)

Better than `AtomicLong` under high contention — uses striped cells to reduce CAS collisions.

```java
LongAdder hits = new LongAdder();
hits.increment();        // thread-safe, low contention
hits.add(5);
long total = hits.sum(); // reads sum across all cells

// LongAccumulator — custom accumulation function
LongAccumulator max = new LongAccumulator(Long::max, Long.MIN_VALUE);
max.accumulate(42);
max.accumulate(17);
long result = max.get(); // 42
```

**Rule of thumb:**
- High contention counters → `LongAdder` (faster than `AtomicLong`)
- CAS-based object swaps → `AtomicReference`
- Low contention or need `compareAndSet` → `AtomicLong` / `AtomicInteger`

### ABA Problem

CAS can falsely succeed: value changes A → B → A between read and CAS.

```java
// Fix: AtomicStampedReference tracks a version stamp alongside the value
AtomicStampedReference<Node> top = new AtomicStampedReference<>(head, 0);

int[] stamp = new int[1];
Node current = top.get(stamp);
top.compareAndSet(current, newNode, stamp[0], stamp[0] + 1); // version check
```

---

## Concurrent Collections

### ConcurrentHashMap

**Java 7:** Divided into 16 segments; each segment is an independent `ReentrantLock`. Reads mostly lock-free; writes lock per segment.

**Java 8+:** No segments. Uses **CAS for empty buckets** and **`synchronized` on the bucket head node** for non-empty buckets. Effectively allows concurrency at individual bucket level — up to `n` (table size) concurrent writers.

```
Internal structure (Java 8):
  Node[] table  (default capacity 16)
  Each slot: null | Node (linked list) | TreeNode (red-black tree when list > 8)
  Resize: triggered at load factor 0.75; uses transfer() with multiple threads helping
```

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// Standard operations — all thread-safe
map.put("key", 1);
map.get("key");
map.remove("key");

// Atomic compound operations
map.putIfAbsent("key", 1);
map.computeIfAbsent("key", k -> expensiveCompute(k));
map.computeIfPresent("key", (k, v) -> v + 1);
map.compute("key", (k, v) -> v == null ? 1 : v + 1); // atomic increment
map.merge("key", 1, Integer::sum);                    // atomic increment (cleaner)

// Bulk operations (parallel, with parallelism threshold)
map.forEach(1, (k, v) -> System.out.println(k + "=" + v));
map.reduceValues(1, v -> v * 2, Integer::sum);
map.search(1, (k, v) -> v > 100 ? k : null);

// Size — approximate under concurrency
int size = map.size();           // not always consistent
long exact = map.mappingCount(); // more accurate for large maps
```

**Key properties:**
- `null` keys and values are **not allowed** (unlike `HashMap`)
- `size()` is approximate — threads can modify during the call
- Iterators are **weakly consistent** — may or may not reflect concurrent modifications
- Read operations (`get`, `containsKey`) are **lock-free**

**CHM vs `Collections.synchronizedMap`:**

| | ConcurrentHashMap | synchronizedMap |
|---|---|---|
| Locking | Per-bucket (fine-grained) | Entire map (coarse) |
| Read performance | Lock-free | Blocks on every read |
| Iteration | Weakly consistent, no lock | Must lock externally |
| Atomic ops | `compute`, `merge`, etc. | Manual synchronized blocks |
| Null keys/values | Not allowed | Allowed |

---

### CopyOnWriteArrayList

On every **write** (add, set, remove), creates a **fresh copy of the entire underlying array**. Readers always see a consistent snapshot — reads are fully lock-free.

```
Internal: volatile Object[] array

write path:
  1. lock (ReentrantLock)
  2. copy array to new array of len+1
  3. apply modification to new array
  4. set volatile reference to new array
  5. unlock

read path:
  1. read volatile reference (no lock)
  2. access element from snapshot — always consistent
```

```java
CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();

list.add("a");
list.addIfAbsent("a"); // only adds if not present

// Safe iteration — snapshot taken at iterator creation; no ConcurrentModificationException
for (String s : list) {
    System.out.println(s); // iterating over snapshot
    list.remove(s);        // safe — modifies new copy, doesn't affect current iterator
}

// Bulk add is more efficient than individual adds (one copy)
list.addAllAbsent(List.of("x", "y", "z"));
```

**When to use:**
- Read-heavy: reads >> writes (event listener lists, observer registries)
- Iteration must never throw `ConcurrentModificationException`
- Small lists (full copy on every write — expensive for large lists)

**When NOT to use:**
- Write-heavy workloads — each write copies the entire array (O(n) memory + time)
- Large lists — copying is expensive

**Trade-offs:**
- Memory: two copies of array exist briefly during write
- Stale reads: iterator sees snapshot, not live data
- No `IndexOutOfBoundsException` during concurrent structural changes

---

### CopyOnWriteArraySet

Backed internally by a `CopyOnWriteArrayList`. Adds set semantics (no duplicates).

```java
CopyOnWriteArraySet<String> set = new CopyOnWriteArraySet<>();

set.add("a");
set.add("a"); // silently ignored — no duplicate

// add() internally calls COWAL.addIfAbsent() — O(n) contains check before every add
```

**Characteristics:**
- `add()` is O(n) — scans entire list for duplicate before inserting
- Iteration and reads are lock-free (same as COWAL)
- Best for very small sets with rare writes (listener sets, role sets)
- **Not suitable** for large sets — O(n) duplicate check + O(n) copy on write

---

### ConcurrentLinkedQueue & Deque

Non-blocking, lock-free linked queue using CAS. Ideal for high-throughput producer-consumer without blocking.

```java
ConcurrentLinkedQueue<Task> queue = new ConcurrentLinkedQueue<>();

queue.offer(task);       // add to tail (never blocks)
Task t = queue.poll();   // remove from head (returns null if empty)
Task peek = queue.peek();// look at head without removing

// ConcurrentLinkedDeque — supports both ends
ConcurrentLinkedDeque<Task> deque = new ConcurrentLinkedDeque<>();
deque.offerFirst(task);
deque.offerLast(task);
deque.pollFirst();
deque.pollLast();
```

**No blocking** — use `BlockingQueue` variants when you need producers/consumers to wait.

---

### BlockingQueue Implementations

`BlockingQueue` adds blocking operations: `put()` blocks if full, `take()` blocks if empty.

```java
// Core methods
queue.put(item);                         // blocks until space available
queue.take();                            // blocks until item available
queue.offer(item, 500, TimeUnit.MS);     // wait up to 500ms
queue.poll(500, TimeUnit.MILLISECONDS);  // wait up to 500ms
queue.offer(item);                       // returns false immediately if full
queue.poll();                            // returns null immediately if empty
```

#### ArrayBlockingQueue

Bounded, backed by array. Optional fairness (FIFO ordering among waiting threads).

```java
BlockingQueue<Task> q = new ArrayBlockingQueue<>(100, true); // capacity=100, fair=true
```

- Fixed capacity — good for backpressure
- Single lock for both head and tail (slightly lower throughput than LinkedBlockingQueue)
- Fair mode prevents starvation but reduces throughput

#### LinkedBlockingQueue

Optionally bounded, backed by linked nodes. Separate locks for head (take) and tail (put) — higher throughput.

```java
BlockingQueue<Task> q = new LinkedBlockingQueue<>(1000); // bounded
BlockingQueue<Task> q = new LinkedBlockingQueue<>();     // unbounded — risk of OOM!
```

- Better throughput than `ArrayBlockingQueue` under high concurrency (two-lock algorithm)
- **Unbounded version can cause OOM** — always set capacity in production

#### PriorityBlockingQueue

Unbounded, priority-ordered. Elements must implement `Comparable` or provide `Comparator`.

```java
BlockingQueue<Task> q = new PriorityBlockingQueue<>(11,
    Comparator.comparingInt(Task::getPriority));

q.put(lowPriTask);
q.put(highPriTask);
q.take(); // returns highest priority task
```

- No capacity limit (unbounded) — monitor memory
- `put()` never blocks (unbounded); `take()` blocks if empty
- Not FIFO — equal priority elements have undefined order

#### SynchronousQueue

Zero capacity — each `put()` must be matched by a `take()` (direct handoff).

```java
BlockingQueue<Task> q = new SynchronousQueue<>();
// Producer blocks until a consumer is ready, and vice versa
```

- Used in `Executors.newCachedThreadPool()` internally
- No buffering — lowest latency handoff
- Good for direct transfer between threads

#### DelayQueue

Holds elements until their delay expires. Elements must implement `Delayed`.

```java
public class DelayedTask implements Delayed {
    private final long executeAt;

    public DelayedTask(long delayMs) {
        this.executeAt = System.currentTimeMillis() + delayMs;
    }

    public long getDelay(TimeUnit unit) {
        return unit.convert(executeAt - System.currentTimeMillis(), TimeUnit.MILLISECONDS);
    }

    public int compareTo(Delayed other) {
        return Long.compare(getDelay(TimeUnit.MS), other.getDelay(TimeUnit.MS));
    }
}

BlockingQueue<DelayedTask> q = new DelayQueue<>();
q.put(new DelayedTask(5000)); // available after 5 seconds
q.take(); // blocks until delay expires
```

#### LinkedTransferQueue (Java 7+)

Like `SynchronousQueue` but with optional buffering. `transfer()` hands off directly if consumer waiting, else enqueues.

```java
TransferQueue<Task> q = new LinkedTransferQueue<>();
q.transfer(task);         // blocks until consumed
q.tryTransfer(task, 1, TimeUnit.SECONDS); // timeout
```

#### Choosing a BlockingQueue

| Queue | Bounded | Ordering | Use case |
|---|---|---|---|
| `ArrayBlockingQueue` | Yes | FIFO | Backpressure, bounded buffer |
| `LinkedBlockingQueue` | Optional | FIFO | High-throughput producer-consumer |
| `PriorityBlockingQueue` | No | Priority | Task scheduling |
| `SynchronousQueue` | 0 | N/A | Direct handoff, thread pool tasks |
| `DelayQueue` | No | Delay | Scheduled task execution |
| `LinkedTransferQueue` | No | FIFO | Low-latency handoff with fallback buffer |

---

## Synchronisers

### CountDownLatch

One-shot barrier — threads wait until a count reaches zero. **Cannot be reset.**

```java
int workerCount = 5;
CountDownLatch latch = new CountDownLatch(workerCount);

// Workers
for (int i = 0; i < workerCount; i++) {
    executor.submit(() -> {
        try { doWork(); }
        finally { latch.countDown(); } // decrement, always in finally
    });
}

// Coordinator waits for all workers
latch.await();                          // blocks until count = 0
latch.await(10, TimeUnit.SECONDS);      // with timeout

// Start gate pattern — make all threads start simultaneously
CountDownLatch startGate = new CountDownLatch(1);
CountDownLatch endGate   = new CountDownLatch(workerCount);

for (int i = 0; i < workerCount; i++) {
    new Thread(() -> {
        try { startGate.await(); doWork(); }
        catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        finally { endGate.countDown(); }
    }).start();
}

startGate.countDown(); // release all threads at once
endGate.await();       // wait for all to finish
```

---

### CyclicBarrier

Reusable barrier — all threads wait until everyone reaches the barrier point. **Can be reset and reused.**

```java
int parties = 4;
CyclicBarrier barrier = new CyclicBarrier(parties, () -> {
    // Barrier action — runs once when all threads arrive, before they're released
    System.out.println("All threads reached checkpoint — proceeding to next phase");
});

Runnable task = () -> {
    try {
        doPhase1();
        barrier.await(); // wait for all threads
        doPhase2();
        barrier.await(); // reusable — wait again for phase 3
        doPhase3();
    } catch (InterruptedException | BrokenBarrierException e) {
        Thread.currentThread().interrupt();
    }
};
```

**CountDownLatch vs CyclicBarrier:**
- `CountDownLatch`: one thread waits for others (or a count of events). Non-reusable.
- `CyclicBarrier`: all threads wait for each other at a common point. Reusable. All participate.

---

### Semaphore

Controls access to a resource pool — limits the number of concurrent accessors.

```java
Semaphore semaphore = new Semaphore(3, true); // 3 permits, fair

public void accessResource() throws InterruptedException {
    semaphore.acquire();        // acquire 1 permit (blocks if none available)
    try {
        useResource();
    } finally {
        semaphore.release();    // always release in finally
    }
}

// Try without blocking
if (semaphore.tryAcquire()) {
    try { useResource(); }
    finally { semaphore.release(); }
}

// Binary semaphore (mutex that can be released by another thread)
Semaphore mutex = new Semaphore(1);
```

**Use cases:** Connection pool throttling, rate limiting, bounded resource access.

---

### Phaser

Flexible, reusable, dynamic barrier — parties can register/deregister. Replaces both `CountDownLatch` and `CyclicBarrier`.

```java
Phaser phaser = new Phaser(1); // 1 = registering the controlling thread

for (int i = 0; i < 3; i++) {
    phaser.register(); // register new party
    new Thread(() -> {
        doWork();
        phaser.arriveAndAwaitAdvance(); // await phase completion
        doMoreWork();
        phaser.arriveAndDeregister();   // done — deregister
    }).start();
}

phaser.arriveAndAwaitAdvance(); // controlling thread waits
phaser.arriveAndDeregister();   // controlling thread exits

// Termination — phaser terminates when parties = 0
// or override onAdvance() to control termination
Phaser limited = new Phaser(workers) {
    @Override
    protected boolean onAdvance(int phase, int registeredParties) {
        return phase >= 2 || registeredParties == 0; // terminate after 3 phases
    }
};
```

---

### Exchanger

Two threads swap objects at a synchronisation point.

```java
Exchanger<DataBuffer> exchanger = new Exchanger<>();

// Thread 1 — producer
DataBuffer filled = fillBuffer();
DataBuffer empty = exchanger.exchange(filled); // swap with consumer
// now has empty buffer to refill

// Thread 2 — consumer
DataBuffer empty = new DataBuffer();
DataBuffer filled = exchanger.exchange(empty); // swap with producer
processBuffer(filled);
```

---

## Executor Framework & Thread Pools

### ThreadPoolExecutor

The core configurable thread pool. All `Executors` factory methods create `ThreadPoolExecutor` internally.

```
ThreadPoolExecutor parameters:
  corePoolSize    — threads kept alive even when idle
  maximumPoolSize — maximum threads allowed
  keepAliveTime   — idle time before excess threads are terminated
  workQueue       — holds tasks before execution
  threadFactory   — creates new threads (for naming, daemon flags)
  rejectionHandler— policy when queue is full and max threads reached
```

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    4,                              // corePoolSize
    8,                              // maximumPoolSize
    60L, TimeUnit.SECONDS,          // keepAliveTime
    new LinkedBlockingQueue<>(200), // bounded work queue
    new ThreadFactoryBuilder()
        .setNameFormat("worker-%d")
        .setDaemon(false)
        .build(),
    new ThreadPoolExecutor.CallerRunsPolicy() // rejection policy
);

executor.prestartAllCoreThreads(); // warm up core threads
executor.allowCoreThreadTimeOut(true); // allow core threads to time out too
```

**Thread creation logic:**
1. If threads < `corePoolSize` → create new thread
2. If threads >= `corePoolSize` → add to work queue
3. If queue is full AND threads < `maximumPoolSize` → create new thread
4. If queue full AND threads = `maximumPoolSize` → apply rejection policy

**Rejection Policies:**

| Policy | Behaviour |
|---|---|
| `AbortPolicy` (default) | Throws `RejectedExecutionException` |
| `CallerRunsPolicy` | Caller thread runs the task (natural backpressure) |
| `DiscardPolicy` | Silently discards the task |
| `DiscardOldestPolicy` | Discards oldest queued task, retries submit |

```java
// Graceful shutdown
executor.shutdown();                              // stop accepting, finish queued tasks
executor.awaitTermination(30, TimeUnit.SECONDS);  // wait for completion
if (!executor.isTerminated()) {
    executor.shutdownNow(); // interrupt running tasks
}
```

---

### Built-in Pool Factories

```java
// Fixed pool — bounded, reuses N threads
ExecutorService fixed = Executors.newFixedThreadPool(8);
// → ThreadPoolExecutor(8, 8, 0, LinkedBlockingQueue(unbounded!))
// ⚠️ Unbounded queue — can cause OOM under high load

// Cached pool — unbounded threads, 60s idle timeout
ExecutorService cached = Executors.newCachedThreadPool();
// → ThreadPoolExecutor(0, MAX_INT, 60s, SynchronousQueue)
// ⚠️ Can create thousands of threads under burst load

// Single thread — serial execution, replaces dead thread
ExecutorService single = Executors.newSingleThreadExecutor();

// Scheduled — run after delay or periodically
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(2);
scheduled.schedule(task, 5, TimeUnit.SECONDS);
scheduled.scheduleAtFixedRate(task, 0, 1, TimeUnit.SECONDS);
scheduled.scheduleWithFixedDelay(task, 0, 1, TimeUnit.SECONDS);

// Virtual threads (Java 21+) — lightweight, millions of threads
ExecutorService virtual = Executors.newVirtualThreadPerTaskExecutor();
```

**Production rule:** Never use `Executors.newFixedThreadPool` or `newCachedThreadPool` in production. Build `ThreadPoolExecutor` directly with bounded queues and explicit rejection policies.

---

### ForkJoinPool

Designed for **divide-and-conquer recursive tasks**. Uses **work-stealing** — idle threads steal tasks from busy threads' queues.

```java
ForkJoinPool pool = new ForkJoinPool(
    Runtime.getRuntime().availableProcessors(),
    ForkJoinPool.defaultForkJoinWorkerThreadFactory,
    null,   // uncaught exception handler
    false   // asyncMode
);

// RecursiveTask — returns a result
public class SumTask extends RecursiveTask<Long> {
    private static final int THRESHOLD = 1000;
    private final long[] array;
    private final int from, to;

    @Override
    protected Long compute() {
        if (to - from <= THRESHOLD) {
            long sum = 0;
            for (int i = from; i < to; i++) sum += array[i];
            return sum;
        }
        int mid = (from + to) / 2;
        SumTask left  = new SumTask(array, from, mid);
        SumTask right = new SumTask(array, mid, to);
        left.fork();                        // async in pool
        return right.compute() + left.join(); // current thread computes right
    }
}

long total = pool.invoke(new SumTask(array, 0, array.length));

// Parallel streams use the common ForkJoinPool internally
long sum = Arrays.stream(largeArray).parallel().sum();

// Custom pool for parallel streams (avoid common pool contention)
ForkJoinPool customPool = new ForkJoinPool(4);
long result = customPool.submit(() ->
    Arrays.stream(largeArray).parallel().sum()
).get();
```

---

### CompletableFuture

Composable, non-blocking async pipelines (Java 8+).

```java
// Basic async
CompletableFuture<User> future = CompletableFuture.supplyAsync(
    () -> userRepo.findById(userId),
    executor // always provide executor — default uses ForkJoinPool.commonPool()
);

// Chaining
CompletableFuture<String> pipeline = CompletableFuture
    .supplyAsync(() -> fetchUser(id), executor)
    .thenApplyAsync(user -> enrichUser(user), executor)    // transform result
    .thenApplyAsync(user -> user.getName(), executor)
    .exceptionally(ex -> "default-name");                  // handle error

// Combining
CompletableFuture<User>    userFuture    = fetchUserAsync(id);
CompletableFuture<Account> accountFuture = fetchAccountAsync(id);

CompletableFuture<Dashboard> dashboard = userFuture
    .thenCombineAsync(accountFuture, (user, account) -> buildDashboard(user, account));

// Run all, wait for all
CompletableFuture.allOf(f1, f2, f3).thenRun(() -> System.out.println("All done"));

// Run all, return first to complete
CompletableFuture.anyOf(f1, f2, f3).thenAccept(result -> use(result));

// Exception handling
future
    .thenApply(result -> transform(result))
    .exceptionally(ex -> fallback(ex))         // recover
    .whenComplete((result, ex) -> log(result, ex)); // always runs

// Timeout (Java 9+)
future.orTimeout(5, TimeUnit.SECONDS)
      .exceptionally(ex -> defaultValue);

future.completeOnTimeout(defaultValue, 5, TimeUnit.SECONDS);
```

---

## Thread Lifecycle & Interruption

### Thread States

```
NEW → RUNNABLE → BLOCKED (waiting for monitor lock)
              → WAITING (Object.wait, Thread.join, LockSupport.park)
              → TIMED_WAITING (sleep, wait(timeout), join(timeout))
              → TERMINATED
```

### Interruption Protocol

Java uses **cooperative interruption** — a thread cannot be forcibly stopped. The interrupt flag is a signal the thread must check and act on.

```java
// Set the interrupt flag
thread.interrupt();

// Check and respond in long-running tasks
public void run() {
    while (!Thread.currentThread().isInterrupted()) {
        doWork();
    }
}

// Blocking operations throw InterruptedException — always restore the flag
public void blockingTask() throws InterruptedException {
    try {
        Thread.sleep(1000);
        queue.take();
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt(); // restore the flag
        throw e; // or handle gracefully
    }
}

// Never swallow InterruptedException silently
catch (InterruptedException e) {
    // ❌ Bad — swallows the signal, caller never knows
    log.warn("Interrupted", e);
}
```

---

## Common Concurrency Problems

### Deadlock

Two threads each hold a lock the other needs.

```java
// ❌ Deadlock — inconsistent lock ordering
void transfer(Account from, Account to, int amount) {
    synchronized (from) {
        synchronized (to) { /* T1: from=A,to=B; T2: from=B,to=A → deadlock */ }
    }
}

// ✅ Fix 1 — consistent lock ordering (by ID)
void transfer(Account from, Account to, int amount) {
    Account first  = from.getId() < to.getId() ? from : to;
    Account second = from.getId() < to.getId() ? to   : from;
    synchronized (first) {
        synchronized (second) { /* always acquire in same order */ }
    }
}

// ✅ Fix 2 — tryLock with timeout
if (lock1.tryLock(100, TimeUnit.MS) && lock2.tryLock(100, TimeUnit.MS)) {
    try { /* ... */ }
    finally { lock1.unlock(); lock2.unlock(); }
}
```

### Livelock

Threads respond to each other's state but make no progress. Fix: introduce randomness or backoff.

### Starvation

A thread never gets CPU time due to priority or unfair locking. Fix: use fair locks (`new ReentrantLock(true)`).

### Race Condition

```java
// ❌ Check-then-act race
if (!map.containsKey(key)) {      // Thread A checks — not present
    map.put(key, compute(key));    // Thread B also checks — also not present; both put
}

// ✅ Atomic
map.computeIfAbsent(key, k -> compute(k));
```

---

## Best Practices & Decision Guide

### Choosing a synchronisation mechanism

```
Need simple counter / flag?
  └── AtomicInteger / AtomicBoolean / volatile

Need compound atomic operation (CAS)?
  └── AtomicReference / AtomicStampedReference

High-contention counter (many threads incrementing)?
  └── LongAdder

Need mutual exclusion?
  └── synchronized (simple, built-in)
  └── ReentrantLock (need tryLock / timeout / Condition)

Read-heavy shared state?
  └── ReadWriteLock (many readers, few writers)
  └── StampedLock (optimistic reads for maximum throughput)

Thread-safe collection?
  └── Map: ConcurrentHashMap
  └── Read-heavy list: CopyOnWriteArrayList
  └── Queue (non-blocking): ConcurrentLinkedQueue
  └── Queue (blocking, bounded): ArrayBlockingQueue / LinkedBlockingQueue
  └── Queue (priority): PriorityBlockingQueue

Waiting for multiple threads?
  └── CountDownLatch (one-shot, one waits for others)
  └── CyclicBarrier (reusable, all wait for each other)
  └── Phaser (dynamic, reusable, multi-phase)

Limit concurrent access to resource?
  └── Semaphore

Async non-blocking pipelines?
  └── CompletableFuture

Divide-and-conquer parallel computation?
  └── ForkJoinPool / RecursiveTask
```

### Thread pool sizing

```
CPU-bound tasks:
  poolSize = Runtime.getRuntime().availableProcessors() + 1

I/O-bound tasks:
  poolSize = cores × (1 + wait_time / compute_time)
  // e.g. 8 cores, 90% wait → 8 × (1 + 9) = 80 threads

Mixed:
  Profile first; start with 2× cores and tune
```

### General rules

- Prefer immutable objects — no synchronisation needed
- Confine mutable state to one thread — no sharing = no contention
- Use `java.util.concurrent` utilities over raw `wait()/notify()`
- Always hold locks for the minimum time
- Never call foreign methods (callbacks, listeners) while holding a lock
- Always shut down `ExecutorService` — it prevents JVM from exiting
- Set bounded queues in production thread pools — unbounded queues hide overload
- Use `ThreadLocal` for per-thread state (e.g., `SimpleDateFormat`, DB connections)

```java
// ThreadLocal pattern
private static final ThreadLocal<SimpleDateFormat> formatter =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

// Always remove to avoid memory leaks in thread pools
try {
    return formatter.get().format(date);
} finally {
    formatter.remove(); // critical in pooled threads
}
```

---

## Quick Reference Cheat Sheet

### Classes at a glance

| Class | Package | Purpose |
|---|---|---|
| `volatile` | keyword | Visibility + ordering, no atomicity |
| `synchronized` | keyword | Mutual exclusion + visibility |
| `ReentrantLock` | j.u.c.locks | Flexible mutex (tryLock, timeout, fair) |
| `ReentrantReadWriteLock` | j.u.c.locks | Multiple readers or one writer |
| `StampedLock` | j.u.c.locks | Optimistic reads, highest throughput |
| `AtomicInteger/Long` | j.u.c.atomic | Lock-free CAS counter |
| `AtomicReference` | j.u.c.atomic | Lock-free object swap |
| `LongAdder` | j.u.c.atomic | High-contention counter |
| `ConcurrentHashMap` | j.u.c | Thread-safe map, fine-grained locking |
| `CopyOnWriteArrayList` | j.u.c | Read-heavy list, COW on write |
| `CopyOnWriteArraySet` | j.u.c | Read-heavy set, backed by COWAL |
| `ArrayBlockingQueue` | j.u.c | Bounded FIFO blocking queue |
| `LinkedBlockingQueue` | j.u.c | Optionally bounded, higher throughput |
| `PriorityBlockingQueue` | j.u.c | Unbounded priority blocking queue |
| `SynchronousQueue` | j.u.c | Zero-capacity direct handoff |
| `CountDownLatch` | j.u.c | One-shot count-down barrier |
| `CyclicBarrier` | j.u.c | Reusable all-parties barrier |
| `Semaphore` | j.u.c | Permit-based resource access control |
| `Phaser` | j.u.c | Dynamic multi-phase barrier |
| `ThreadPoolExecutor` | j.u.c | Configurable thread pool |
| `ForkJoinPool` | j.u.c | Work-stealing divide-and-conquer pool |
| `CompletableFuture` | j.u.c | Async composable pipeline |
| `ThreadLocal` | java.lang | Per-thread variable storage |

### Memory visibility quick rules

| Mechanism | Visibility | Atomicity | Ordering |
|---|---|---|---|
| `volatile` | ✅ | ❌ (compound) | ✅ |
| `synchronized` | ✅ | ✅ | ✅ |
| `AtomicXxx` | ✅ | ✅ (CAS) | ✅ |
| `final` field | ✅ (after ctor) | N/A | ✅ |
| No sync | ❌ | ❌ | ❌ |

---

*References: Brian Goetz — Java Concurrency in Practice (2006) | JSR-133 Java Memory Model | OpenJDK source (java.util.concurrent)*
