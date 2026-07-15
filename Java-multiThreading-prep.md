# Java Multithreading & Concurrency Interview Prep (What / Why / How + Example)

---

## Section 1: Threading Fundamentals

**1. Process vs Thread?**
- **Process** — an independent running program with its own memory space, isolated from other processes.
- **Thread** — a lightweight unit of execution *within* a process, sharing the process's memory (heap) but with its own stack.
- **Why threads matter:** cheaper to create/switch than processes, and shared memory makes communication between them fast (but also dangerous — that's where most concurrency bugs come from).

**2. How do you create a thread in Java? Two ways.**
```java
// 1. Extend Thread
class MyThread extends Thread {
    public void run() { System.out.println("running"); }
}
new MyThread().start();

// 2. Implement Runnable (preferred)
Runnable task = () -> System.out.println("running");
new Thread(task).start();
```
- **Why `Runnable` is preferred:** Java doesn't support multiple class inheritance, so extending `Thread` uses up your one inheritance slot. Implementing `Runnable` keeps that flexibility open, and works directly with `ExecutorService`.

**3. `start()` vs `run()` — what's the difference?**
- `start()` — creates a **new thread** and calls `run()` on it asynchronously.
- `run()` — just a normal method call, executes on the **current thread**, no new thread created.
- **Common mistake:** calling `thread.run()` directly runs it synchronously on the caller's thread — no concurrency happens at all.

**4. What are Thread States (lifecycle)?**
`NEW` → `RUNNABLE` → (`BLOCKED` / `WAITING` / `TIMED_WAITING`) → `TERMINATED`. A thread moves to `BLOCKED` waiting for a lock, `WAITING`/`TIMED_WAITING` when it calls `wait()`/`join()`/`sleep()`.

**5. What is a Daemon thread?**
- **What:** A background thread that doesn't prevent the JVM from exiting (e.g., garbage collector thread). JVM exits once only daemon threads remain.
```java
Thread t = new Thread(task);
t.setDaemon(true);
t.start();
```

**6. `sleep()` vs `wait()`?**
- `Thread.sleep(ms)` — pauses the current thread, **does NOT release any lock** it holds.
- `Object.wait()` — releases the lock and pauses until `notify()`/`notifyAll()` is called (or timeout) — used for inter-thread signaling, must be called inside a `synchronized` block.

**7. What is `join()` used for?**
- **What:** Makes the calling thread wait until the target thread finishes execution.
- **Why:** Useful when you need results from a thread before proceeding.
```java
Thread t = new Thread(() -> processPayroll());
t.start();
t.join(); // main thread waits here until t finishes
System.out.println("Payroll done");
```

**8. What is a Race Condition?**
- **What:** When multiple threads access/modify shared mutable data concurrently, and the final outcome depends on unpredictable timing/interleaving of operations.
- **Example:**
```java
class Counter {
    int count = 0;
    void increment() { count++; } // NOT atomic! read-modify-write = 3 steps
}
// Two threads calling increment() 1000 times each might NOT result in count == 2000
```

**9. What is Thread Safety?**
- Code that behaves correctly when accessed by multiple threads simultaneously, without producing race conditions or corrupted state — achieved via synchronization, atomic variables, or immutability.

**10. What is Context Switching and why is it costly?**
- **What:** The CPU switching from executing one thread to another (saving/restoring register state, stack pointer, etc.).
- **Why costly:** Pure overhead — no useful work is done during the switch itself, and too many threads competing for few CPU cores leads to excessive switching (thrashing), hurting throughput.

---

## Section 2: `synchronized`

**11. What is `synchronized` and what problem does it solve?**
- **What:** A keyword that ensures only **one thread at a time** can execute a block/method, by acquiring an intrinsic lock (monitor) on an object.
- **Why:** Prevents race conditions on shared mutable state — fixes the `Counter` example from Q8.
```java
class Counter {
    int count = 0;
    synchronized void increment() { count++; } // now atomic w.r.t. other synchronized calls
}
```

**12. `synchronized` method vs `synchronized` block — difference?**
- **Method-level** — locks on `this` (or the class object, for static methods) for the *entire* method.
- **Block-level** — locks only the critical section you specify, on an object of your choice — more fine-grained, better performance since less code is locked.
```java
void updateBalance(double amt) {
    // non-critical work here, no lock needed
    synchronized (this) {
        balance += amt; // only this part needs protection
    }
}
```

**13. What is the difference between locking on `this` vs a static/class-level object?**
- Instance-level `synchronized` locks per-object — two different instances don't block each other.
- `static synchronized` methods lock on the `Class` object itself — shared across **all instances**, since static state is shared class-wide.
```java
static synchronized void staticMethod() { ... } // locks on MyClass.class
```

**14. What is a Monitor / Intrinsic Lock in Java?**
- Every Java object has an implicit lock (monitor) associated with it. `synchronized` acquires this monitor on entry and releases it on exit (even if an exception is thrown) — this is why `synchronized` is reentrant-safe and exception-safe by default.

**15. Is `synchronized` reentrant? What does that mean?**
- **Yes.** A thread already holding a lock can re-acquire it (e.g., calling another synchronized method on the same object from within a synchronized method) without deadlocking itself.
```java
synchronized void outer() { inner(); } // same thread, same lock — fine
synchronized void inner() { ... }
```

**16. What's the downside of using `synchronized` everywhere?**
- Reduces concurrency (only one thread proceeds at a time through that section), can lead to contention/performance bottlenecks, and coarse-grained locking (locking too much code) makes it worse. Doesn't offer timeout or interruptibility like `ReentrantLock` does.

**17. `synchronized` vs `ReentrantLock` — when would you use `ReentrantLock`?**
- `ReentrantLock` (from `java.util.concurrent.locks`) offers more control: `tryLock()` with timeout (avoid waiting forever), interruptible lock acquisition, and fairness policies (FIFO ordering of waiting threads) — `synchronized` has none of these.
```java
ReentrantLock lock = new ReentrantLock();
if (lock.tryLock(2, TimeUnit.SECONDS)) {
    try { /* critical section */ }
    finally { lock.unlock(); } // MUST unlock manually, unlike synchronized
}
```

---

## Section 3: `volatile`

**18. What is `volatile` and what guarantee does it give?**
- **What:** A keyword ensuring a variable's reads/writes always go directly to **main memory**, not a thread's local CPU cache/register.
- **Why:** Without it, one thread's update to a variable might not be "visible" to another thread promptly (or ever, in the worst case) due to CPU caching and compiler reordering optimizations.
```java
class FlagHolder {
    volatile boolean running = true;
    void stop() { running = false; } // visible to other threads immediately
}
```

**19. Does `volatile` guarantee atomicity? Common trap question.**
- **No!** `volatile` only guarantees **visibility**, not atomicity. Compound operations like `count++` (read-modify-write) are still NOT thread-safe even if `count` is `volatile`, because the operation itself isn't a single atomic step.
```java
volatile int count = 0;
count++; // STILL a race condition — read, increment, write are 3 separate steps
```

**20. When is `volatile` sufficient on its own (without `synchronized`)?**
- When the variable is a simple flag or single reference being read/written as a whole, with no compound read-modify-write logic — e.g., a `volatile boolean shutdownRequested` flag checked in a loop by a worker thread.

**21. What is the Java Memory Model (JMM) and why does it matter for `volatile`?**
- **What:** The JMM defines the rules for how/when changes made by one thread become visible to others, and what reorderings the compiler/CPU are allowed to do.
- **Why relevant to `volatile`:** `volatile` establishes a **happens-before** relationship — a write to a volatile variable happens-before every subsequent read of it by another thread, preventing certain instruction reordering around it.

**22. `volatile` vs `synchronized` — key differences?**
| | `volatile` | `synchronized` |
|---|---|---|
| Guarantees | Visibility only | Visibility + Atomicity (mutual exclusion) |
| Blocking | Never blocks | Blocks other threads |
| Use case | Simple flags/single values | Compound operations/critical sections |

---

## Section 4: Atomic Classes & Atomicity

**23. What does "Atomicity" actually mean?**
- **What:** An operation that appears to happen instantaneously, as a single indivisible step — no other thread can observe it "half-done."
- **Why it matters:** `count++` looks like one step in code but is actually 3 (read, add, write) at the machine level — a thread can be interrupted mid-way, causing lost updates.

**24. What are Atomic classes (`AtomicInteger`, `AtomicLong`, etc.) and how do they achieve atomicity without locks?**
- **What:** Classes in `java.util.concurrent.atomic` providing lock-free thread-safe operations on single variables.
- **How:** Internally use **CAS (Compare-And-Swap)**, a hardware-supported atomic instruction — "if the current value still equals what I expect, swap it for the new value; otherwise retry" — avoiding the overhead of acquiring a lock.
```java
AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet(); // atomic, no synchronized needed
```

**25. What is CAS (Compare-And-Swap) in detail?**
- **How it works:** `compareAndSet(expectedValue, newValue)` — the CPU atomically checks if the current value matches `expectedValue`; if yes, updates to `newValue` and returns true; if another thread changed it in between, it returns false and the caller typically retries in a loop (spin).
- **Why it's often faster than locks:** No thread ever blocks/sleeps waiting for a lock — under low contention, this is much cheaper than the overhead of `synchronized`'s lock acquisition.

**26. When would `AtomicInteger` NOT be enough, and you'd need `synchronized`/`ReentrantLock` instead?**
- When you need to atomically update **multiple related variables together** (e.g., transferring money — debit one account, credit another) — atomic classes only guarantee atomicity for a *single* variable's operation, not a compound multi-variable transaction.

**27. What is `AtomicReference` used for?**
- Same CAS-based atomicity, but for object references instead of primitives — useful for lock-free updates to an entire object (e.g., swapping in an updated immutable config object).

**28. What is the ABA problem in CAS-based algorithms?**
- **What:** A value changes from A → B → back to A between a thread's read and its CAS check — CAS sees "still A" and proceeds, even though the value *did* change in between (which might matter if B's change had side effects).
- **How it's mitigated:** `AtomicStampedReference` — pairs the value with a version/stamp that increments on every change, so CAS also checks the stamp, not just the value.

---

## Section 5: Comparator vs Comparable

**29. `Comparable` vs `Comparator` — core difference?**
- **`Comparable`** — defines a class's **natural/default** ordering, implemented **inside** the class via `compareTo()`. A class can only have one natural ordering.
- **`Comparator`** — defines **custom, external** ordering logic, separate from the class. You can define many different `Comparator`s for the same class (sort by name, by salary, by date, etc.).

**30. `Comparable` — example.**
```java
class Employee implements Comparable<Employee> {
    String name;
    double salary;

    @Override
    public int compareTo(Employee other) {
        return Double.compare(this.salary, other.salary); // natural order: by salary ascending
    }
}
Collections.sort(employees); // uses compareTo() automatically
```

**31. `Comparator` — example, including multi-field sorting.**
```java
Comparator<Employee> byName = (e1, e2) -> e1.getName().compareTo(e2.getName());

// Modern style with Comparator.comparing + chaining
Comparator<Employee> byDeptThenSalaryDesc = Comparator
    .comparing(Employee::getDept)
    .thenComparing(Employee::getSalary, Comparator.reverseOrder());

employees.sort(byDeptThenSalaryDesc);
```

**32. What must `compareTo()`/`compare()` return, and what do the values mean?**
- Negative if the first object is "less than" the second, zero if equal, positive if "greater than." Used by `Collections.sort()`, `TreeMap`, `TreeSet`, etc., to establish ordering.

**33. Why must `compareTo()` be consistent with `equals()` (ideally)?**
- **Why:** Sorted collections like `TreeSet`/`TreeMap` use `compareTo()` (not `equals()`) to determine uniqueness. If `compareTo()` returns 0 for two "different" objects that aren't actually `.equals()`, one will silently be treated as a duplicate and dropped from a `TreeSet` — a subtle, hard-to-find bug.

**34. Can you use `Comparator` and `Comparable` together?**
- Yes — a class can implement `Comparable` for its default order, but you can still pass a different `Comparator` to `sort()` when you need a different order for a specific use case, without changing the class itself.

---

## Section 6: Executor Framework (`ExecutorService`)

**35. What is `ExecutorService` and why use it instead of raw `Thread`s?**
- **What:** A higher-level abstraction for managing a pool of reusable threads.
- **Why:** Creating a new `Thread` per task is expensive (OS-level thread creation) and gives you no control over concurrency limits. `ExecutorService` reuses threads, queues excess tasks, and centralizes lifecycle management (shutdown, monitoring).
```java
ExecutorService pool = Executors.newFixedThreadPool(4);
pool.submit(() -> processPayroll());
pool.shutdown(); // stops accepting new tasks, lets running ones finish
```

**36. Common `Executors` factory methods — what and when to use each?**
- `newFixedThreadPool(n)` — fixed number of threads, good for known/steady workloads.
- `newCachedThreadPool()` — creates threads as needed, reuses idle ones, unbounded — risky under heavy load (can spawn unlimited threads).
- `newSingleThreadExecutor()` — one thread, tasks run sequentially in order — good when you need strict ordering.
- `newScheduledThreadPool(n)` — for delayed/periodic tasks.
- **Modern note:** In production, it's often recommended to construct `ThreadPoolExecutor` directly (rather than `Executors.*`) to explicitly control queue size and avoid unbounded queues/thread counts that `Executors` factories can hide.

**37. `submit()` vs `execute()`?**
- `execute(Runnable)` — fire-and-forget, no result, no way to check for exceptions afterward.
- `submit(Callable/Runnable)` — returns a `Future`, lets you retrieve a result (`Callable`) or check completion/exceptions later.

**38. What is `Future` and how do you use it?**
```java
Callable<Integer> task = () -> computeLOP(employeeId);
Future<Integer> future = executor.submit(task);
Integer result = future.get(); // blocks until the result is ready (or throws if task failed)
```
- **Why:** Lets the calling thread do other work while the task runs in the background, then collect the result when needed.

**39. What is `CompletableFuture` and how is it different from `Future`?**
- **What:** A more powerful async construct — supports chaining callbacks (`thenApply`, `thenCombine`), composing multiple async operations, and doesn't force you to block with `.get()`.
```java
CompletableFuture.supplyAsync(() -> fetchEmployee(id))
    .thenApply(emp -> emp.getSalary())
    .thenAccept(salary -> System.out.println("Salary: " + salary));
```
- **Why it's preferred over plain `Future`:** `Future.get()` blocks; `CompletableFuture` lets you build non-blocking pipelines and combine multiple async results declaratively.

**40. What happens if a task submitted to an `ExecutorService` throws an exception?**
- With `execute()` — the exception propagates to the thread's uncaught exception handler (often silently logged, easy to miss).
- With `submit()` — the exception is captured inside the `Future`; it's only thrown when you call `future.get()`, which wraps it in an `ExecutionException`.

**41. `shutdown()` vs `shutdownNow()`?**
- `shutdown()` — graceful: stops accepting new tasks, lets already-submitted/running tasks finish.
- `shutdownNow()` — attempts to forcibly stop all actively executing tasks (via interruption) and returns the list of tasks that were still waiting in the queue, never started.

**42. What is a `ThreadPoolExecutor`'s core config: core pool size, max pool size, queue?**
- **Core pool size** — minimum threads kept alive even if idle. **Max pool size** — upper limit threads can grow to under load. **Work queue** — holds tasks waiting for a free thread once core threads are busy; only grows beyond core size once the queue is full too. Misconfiguring these (e.g., unbounded queue) can hide backpressure problems until you run out of memory.

---

## Section 7: Deadlock, Livelock & Related Problems

**43. What is Deadlock? Give a classic example.**
- **What:** Two or more threads waiting forever for locks held by each other, so none can proceed.
```java
// Thread 1: locks A, then tries to lock B
// Thread 2: locks B, then tries to lock A
// Both wait forever
synchronized (lockA) {
    synchronized (lockB) { ... } // Thread 1
}
synchronized (lockB) {
    synchronized (lockA) { ... } // Thread 2 — deadlock if interleaved this way
}
```

**44. How do you prevent deadlock?**
- **Consistent lock ordering** — always acquire locks in the same global order across all threads (fixes the example above).
- Use `tryLock()` with a timeout instead of blocking indefinitely.
- Minimize nested locks / hold locks for the shortest time possible.

**45. What is Livelock (different from deadlock)?**
- **What:** Threads are NOT blocked/waiting — they're actively running, but keep responding to each other in a way that prevents any actual progress (like two people repeatedly stepping aside for each other in a hallway, both trying to be polite, neither passing).

**46. What is Thread Starvation?**
- **What:** A thread never gets CPU time or a lock because other (often higher-priority) threads keep getting preference — it's "alive" but never progresses.

---

## Section 8: Concurrent Collections & Coordination Tools

**47. `ArrayList` vs `CopyOnWriteArrayList`?**
- `CopyOnWriteArrayList` — every write creates a new copy of the underlying array; reads never block/see a "torn" state. Great for read-heavy, rarely-written concurrent scenarios (e.g., a list of event listeners), bad for write-heavy use (copying is expensive).

**48. `HashMap` vs `ConcurrentHashMap`?**
- `ConcurrentHashMap` allows safe concurrent reads/writes without locking the entire map — uses finer-grained internal locking (bucket-level in modern versions), so multiple threads can operate on different parts simultaneously, unlike a fully `synchronized` `HashMap` wrapper which locks the whole map for every operation.

**49. What is `CountDownLatch` and a real use case?**
- **What:** A counter that threads can wait on (`await()`) until it reaches zero, decremented via `countDown()`. **One-time use** — can't be reset.
- **Use case:** Main thread waits for N worker threads to finish initialization before proceeding.
```java
CountDownLatch latch = new CountDownLatch(3);
for (int i = 0; i < 3; i++) {
    new Thread(() -> { doWork(); latch.countDown(); }).start();
}
latch.await(); // blocks until all 3 workers call countDown()
```

**50. What is `CyclicBarrier` and how is it different from `CountDownLatch`?**
- **What:** Makes a fixed number of threads all wait for each other at a common barrier point before any proceed — unlike `CountDownLatch`, it's **reusable** (resets automatically once the barrier is tripped) and threads wait on *each other*, not on an external count.

**51. What is a `Semaphore`?**
- **What:** Controls access to a limited number of "permits" — threads acquire a permit before proceeding, release it when done; if no permits are available, the thread blocks.
- **Use case:** Limiting concurrent access to a resource pool (e.g., only allow 5 concurrent DB connections/API calls at once).
```java
Semaphore semaphore = new Semaphore(5);
semaphore.acquire();
try { callExternalApi(); }
finally { semaphore.release(); }
```

---

## Section 9: Executable Code — Common Interview Coding Problems

**52. Write thread-safe code for a shared counter (two ways).**
```java
// Way 1: synchronized
class SafeCounter {
    private int count = 0;
    synchronized void increment() { count++; }
    synchronized int get() { return count; }
}

// Way 2: AtomicInteger (preferred for simple counters — lock-free, faster under contention)
class SafeCounter {
    private final AtomicInteger count = new AtomicInteger(0);
    void increment() { count.incrementAndGet(); }
    int get() { return count.get(); }
}
```

**53. Implement the Producer-Consumer pattern using `wait()`/`notify()`.**
```java
class SharedQueue {
    private final Queue<Integer> queue = new LinkedList<>();
    private final int capacity = 5;

    synchronized void produce(int value) throws InterruptedException {
        while (queue.size() == capacity) wait(); // full, wait for consumer
        queue.add(value);
        notifyAll(); // wake up any waiting consumer
    }

    synchronized int consume() throws InterruptedException {
        while (queue.isEmpty()) wait(); // empty, wait for producer
        int value = queue.poll();
        notifyAll(); // wake up any waiting producer
        return value;
    }
}
```
- **Why `while` not `if` for the wait condition:** Spurious wakeups can occur (a thread wakes up without an actual `notify()`), so you must re-check the condition in a loop, not assume it's true just because `wait()` returned.

**54. Producer-Consumer using `BlockingQueue` (simpler, modern approach).**
```java
BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(5);

// Producer
new Thread(() -> {
    try { queue.put(42); } catch (InterruptedException e) { }
}).start();

// Consumer
new Thread(() -> {
    try { int val = queue.take(); } catch (InterruptedException e) { }
}).start();
```
- **Why prefer this:** `BlockingQueue` handles all the waiting/notifying internally — no manual `wait()`/`notify()` bugs to worry about.

**55. Write code demonstrating a race condition, then fix it.**
```java
// BUGGY
class BankAccount {
    private double balance = 1000;
    void withdraw(double amt) {
        if (balance >= amt) {         // check
            balance -= amt;            // then act — NOT atomic together!
        }
    }
}
// Two threads withdrawing 800 each from a 1000 balance could both pass the check
// before either updates balance -> balance goes negative

// FIXED
class BankAccount {
    private double balance = 1000;
    synchronized void withdraw(double amt) {
        if (balance >= amt) {
            balance -= amt;
        }
    }
}
```

**56. How would you run 5 tasks in parallel and wait for all of them to complete, collecting results?**
```java
ExecutorService executor = Executors.newFixedThreadPool(5);
List<Callable<Integer>> tasks = List.of(
    () -> computeLOP(1), () -> computeLOP(2), () -> computeLOP(3),
    () -> computeLOP(4), () -> computeLOP(5)
);
List<Future<Integer>> results = executor.invokeAll(tasks); // blocks until ALL complete
for (Future<Integer> f : results) {
    System.out.println(f.get());
}
executor.shutdown();
```

---

## Quick tips for your interviews
- **Q19 (`volatile` doesn't give atomicity) is the #1 trap question** at this level — interviewers love asking "is `volatile int count; count++;` thread-safe?" specifically to see if you understand the difference between visibility and atomicity. Know this cold.
- **Q29-33 (Comparable vs Comparator)** often gets combined with a live coding ask — be ready to sort a `List<Employee>` by multiple fields using `Comparator.comparing().thenComparing()` without hesitation.
- **Q24-25 (Atomic/CAS)** — if asked "how does `AtomicInteger` avoid using locks," explaining CAS at a conceptual level (not needing assembly-level detail) shows real depth beyond "it's just thread-safe."
- Tie your rate-limiter (`AtomicInteger`/`ConcurrentHashMap` for counters) and idempotency (compound check-then-act needing synchronization, Q55's pattern) work directly into these answers — you've already built and debugged exactly these problems in production.

Want a mock round where I fire questions from this doc plus your Stream API and Core Java docs together, simulating a real technical round?
