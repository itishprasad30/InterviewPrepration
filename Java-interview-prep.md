# Java Interview Prep — 50+ Questions (What / Why / How + Example)

Format for each: **What** it is, **Why** it matters/exists, **How** to use it, with a code example.

---

## Section 1: OOP Fundamentals

**1. What is OOP and its four pillars?**
- **What:** A programming paradigm organizing code around objects (data + behavior) instead of functions.
- **Why:** Makes large codebases modular, reusable, and maintainable.
- **How:** Encapsulation, Inheritance, Polymorphism, Abstraction.

**2. What is Encapsulation?**
- **What:** Bundling data (fields) and methods together, hiding internal state behind getters/setters.
- **Why:** Protects data integrity — outside code can't directly corrupt internal state.
- **Example:**
```java
public class Employee {
    private double salary; // hidden
    public double getSalary() { return salary; }
    public void setSalary(double s) {
        if (s < 0) throw new IllegalArgumentException("negative salary");
        this.salary = s;
    }
}
```

**3. What is Inheritance?**
- **What:** A class acquiring fields/methods of another class.
- **Why:** Code reuse, establishes "is-a" relationships.
- **Example:**
```java
class Employee { void work() { System.out.println("working"); } }
class Manager extends Employee { void approveLeave() { ... } }
```

**4. What is Polymorphism? Compile-time vs runtime.**
- **What:** Same interface, different implementations. **Compile-time** = method overloading (same name, different params). **Runtime** = method overriding (subclass changes parent behavior), resolved via dynamic dispatch.
- **Why:** Lets you write generic code that works across different object types.
- **Example:**
```java
class Shape { double area() { return 0; } }
class Circle extends Shape { double area() { return 3.14 * r * r; } }
Shape s = new Circle(); 
s.area(); // calls Circle's version at runtime — runtime polymorphism
```

**5. What is Abstraction?**
- **What:** Hiding implementation details, exposing only what's necessary via abstract classes/interfaces.
- **Why:** Decouples "what to do" from "how it's done" — lets you swap implementations freely.
- **Example:** An `PaymentGateway` interface with `pay()` — you don't care if it's Razorpay or Stripe underneath.

**6. Abstract class vs Interface — when to use which?**
- **What:** Abstract class can have state + partial implementation; interface (pre-Java 8) was pure contract, now can have `default`/`static` methods too.
- **Why the distinction matters:** A class can extend only **one** abstract class but implement **multiple** interfaces.
- **How to choose:** Use abstract class when subclasses share common state/code. Use interface for a pure capability contract (e.g., `Comparable`, `Runnable`).

**7. What is method overloading vs overriding?**
- **Overloading:** Same method name, different parameter list, same class — resolved at compile time.
- **Overriding:** Subclass redefines a parent method with the *same* signature — resolved at runtime.

**8. What is `super` used for?**
- **What:** Refers to the immediate parent class.
- **Why:** Call parent constructor (`super()`), or parent's overridden method (`super.method()`).

**9. Can a constructor be inherited?**
- **No** — constructors aren't inherited, but a subclass constructor implicitly/explicitly calls the parent's via `super()`.

**10. What is the `final` keyword's three uses?**
- `final` variable → constant, can't be reassigned.
- `final` method → can't be overridden.
- `final` class → can't be extended (e.g., `String`).

---

## Section 2: Java Collections Framework

**11. Array vs ArrayList?**
- Array is fixed size, can hold primitives. `ArrayList` is dynamic (resizes), only holds objects (autoboxing for primitives).

**12. `ArrayList` vs `LinkedList`?**
- **What:** `ArrayList` backed by a dynamic array; `LinkedList` backed by doubly-linked nodes.
- **Why it matters:** `ArrayList` → O(1) random access, O(n) insert/delete in middle. `LinkedList` → O(n) access, O(1) insert/delete once you have the node reference.
- **How to choose:** Frequent random reads → `ArrayList`. Frequent insert/delete at ends → `LinkedList`/`ArrayDeque`.

**13. How does `HashMap` work internally?**
- **What:** Array of "buckets," each holding a linked list (or, since Java 8, a red-black tree if a bucket gets too crowded — 8+ entries).
- **How:** `key.hashCode()` → hashed & spread → maps to bucket index via `(n-1) & hash`. On collision, entries in the same bucket are chained (or treeified).
- **Why treeification (Java 8):** Protects against worst-case O(n) lookups if many keys collide (e.g., malicious hash flooding) — treeified buckets give O(log n).
- **Example:**
```java
Map<String, Employee> map = new HashMap<>();
map.put("E101", emp); // hash("E101") -> bucket index -> stored there
```

**14. What's the default capacity and load factor of `HashMap`?**
- Default capacity 16, load factor 0.75. When size exceeds `capacity * loadFactor`, the map resizes (doubles) and rehashes all entries.

**15. Why must `hashCode()` and `equals()` be overridden together?**
- **Why:** `HashMap`/`HashSet` use `hashCode()` to find the bucket, then `equals()` to check actual equality within that bucket. If you override one without the other, objects that are "equal" might land in different buckets and never be found as duplicates.
- **Example:**
```java
class Employee {
    String id;
    @Override public boolean equals(Object o) {
        return o instanceof Employee && ((Employee)o).id.equals(this.id);
    }
    @Override public int hashCode() { return id.hashCode(); }
}
```

**16. `HashMap` vs `ConcurrentHashMap`?**
- `HashMap` is not thread-safe — concurrent modification can corrupt internal structure or infinite-loop (pre-Java 8 resize bug). `ConcurrentHashMap` allows safe concurrent access by locking at a finer granularity (per-bucket/segment, not the whole map), so multiple threads can read/write different parts simultaneously.

**17. `HashMap` vs `Hashtable`?**
- `Hashtable` is legacy, synchronized on every method (slow), doesn't allow null key/values. `HashMap` allows one null key, is unsynchronized, and is generally preferred (use `ConcurrentHashMap` if thread safety is needed).

**18. `HashMap` vs `TreeMap` vs `LinkedHashMap`?**
- `HashMap`: no ordering guarantee, O(1) average access.
- `LinkedHashMap`: maintains insertion order.
- `TreeMap`: sorted by keys (natural order or custom `Comparator`), O(log n) operations (red-black tree).

**19. What is `HashSet` internally?**
- Backed by a `HashMap` under the hood — elements are stored as keys with a dummy constant value.

**20. `Comparable` vs `Comparator`?**
- **`Comparable`** — defines a class's *natural* ordering, implemented inside the class itself (`compareTo`).
- **`Comparator`** — external, defines custom ordering, can have multiple for the same class.
- **Example:**
```java
// Comparable
class Employee implements Comparable<Employee> {
    int salary;
    public int compareTo(Employee o) { return this.salary - o.salary; }
}
// Comparator
Comparator<Employee> byName = (e1, e2) -> e1.name.compareTo(e2.name);
employees.sort(byName);
```

**21. Fail-fast vs fail-safe iterators?**
- **Fail-fast** (`ArrayList`, `HashMap`) — throws `ConcurrentModificationException` if the collection is structurally modified while iterating.
- **Fail-safe** (`CopyOnWriteArrayList`, `ConcurrentHashMap`) — iterates over a snapshot/copy, doesn't throw, but may not reflect the latest changes.

**22. What is `Iterator` vs `ListIterator`?**
- `Iterator` — forward-only traversal, works on any `Collection`.
- `ListIterator` — bidirectional, allows adding/setting elements during iteration, only for `List`.

**23. What's the difference between `List`, `Set`, `Map`?**
- `List` — ordered, allows duplicates.
- `Set` — no duplicates.
- `Map` — key-value pairs, keys unique.

**24. How do you make a collection thread-safe?**
- `Collections.synchronizedList(list)`, or use concurrent collections (`ConcurrentHashMap`, `CopyOnWriteArrayList`) which are more scalable than a global lock.

---

## Section 3: Java 8+ Features

**25. What are Lambda expressions?**
- **What:** Anonymous functions — a concise way to implement functional interfaces (interfaces with exactly one abstract method).
- **Why:** Removes boilerplate of anonymous inner classes; enables functional-style code.
- **Example:**
```java
// Before Java 8
Runnable r = new Runnable() { public void run() { System.out.println("hi"); } };
// Java 8
Runnable r2 = () -> System.out.println("hi");
```

**26. What is a Functional Interface?**
- **What:** An interface with exactly one abstract method (can have default/static methods too), annotated with `@FunctionalInterface` (optional but recommended).
- **Examples:** `Runnable`, `Comparator<T>`, `Function<T,R>`, `Supplier<T>`, `Predicate<T>`, `Consumer<T>`.

**27. What is the Streams API?**
- **What:** A pipeline for processing collections declaratively (filter/map/reduce) instead of imperative loops.
- **Why:** More readable, composable, and (when needed) parallelizable.
- **Example:**
```java
List<String> names = employees.stream()
    .filter(e -> e.getSalary() > 50000)
    .map(Employee::getName)
    .collect(Collectors.toList());
```

**28. `map()` vs `flatMap()`?**
- `map()` — transforms each element 1-to-1. `flatMap()` — transforms each element into a stream, then flattens all those streams into one (useful for nested collections, e.g., `List<List<String>>` → `List<String>`).

**29. What is `Optional` and why use it?**
- **What:** A container that may or may not hold a value, avoiding raw `null`.
- **Why:** Forces explicit handling of "value might be absent," reducing `NullPointerException`s.
- **Example:**
```java
Optional<Employee> emp = repo.findById(id);
emp.ifPresentOrElse(
    e -> System.out.println(e.getName()),
    () -> System.out.println("not found")
);
```

**30. What are default and static methods in interfaces?**
- **What:** Java 8 allows interfaces to have method bodies (`default`) or utility methods (`static`).
- **Why:** Lets you add new methods to interfaces without breaking existing implementing classes (this is literally why `Stream` methods like `forEach` were added to `Collection` without breaking every existing class).

**31. What is Method Reference?**
- **What:** Shorthand for a lambda that just calls an existing method — `ClassName::methodName`.
- **Example:** `employees.forEach(System.out::println);`

**32. What's new in `String` handling in modern Java (text blocks)?**
- Java 13+ introduced text blocks (`"""..."""`) for multi-line strings without escape characters — useful for JSON/HTML templates.

---

## Section 4: Exception Handling

**33. Checked vs Unchecked exceptions?**
- **Checked** (e.g., `IOException`) — must be declared/caught at compile time, represent recoverable conditions.
- **Unchecked** (`RuntimeException` and subclasses, e.g., `NullPointerException`) — not enforced by compiler, usually programming bugs.

**34. `throw` vs `throws`?**
- `throw` actually throws an exception instance. `throws` is in a method signature, declaring what checked exceptions it might propagate.

**35. What is try-with-resources?**
- **What:** Auto-closes resources (`AutoCloseable`) at the end of the block, even on exception.
- **Why:** Avoids resource leaks (previously needed verbose `finally { conn.close(); }`).
- **Example:**
```java
try (Connection conn = dataSource.getConnection()) {
    // conn auto-closed here
} catch (SQLException e) { ... }
```

**36. What happens if you have a `return` in both `try` and `finally`?**
- The `finally` block's return **overrides** the try block's return — a classic gotcha (avoid returning from `finally`).

**37. Custom exceptions — why and how?**
- **Why:** Domain-specific errors carry more meaning than generic exceptions (e.g., `EmployeeNotFoundException` vs generic `RuntimeException`).
```java
public class EmployeeNotFoundException extends RuntimeException {
    public EmployeeNotFoundException(String msg) { super(msg); }
}
```

---

## Section 5: Multithreading & Concurrency

**38. Process vs Thread?**
- A process is an independent program with its own memory space. A thread is a lightweight unit of execution *within* a process, sharing the process's memory.

**39. How do you create a thread in Java?**
- Extend `Thread`, or implement `Runnable` (preferred — allows extending other classes too, and works better with `ExecutorService`).
```java
Runnable task = () -> System.out.println("running");
new Thread(task).start();
```

**40. What is `synchronized`?**
- **What:** Keyword ensuring only one thread executes a block/method at a time, preventing race conditions.
- **Why:** Protects shared mutable state.
```java
public synchronized void updateBalance(double amt) { balance += amt; }
```

**41. What is a Race Condition?**
- **What:** When multiple threads access/modify shared data concurrently and the outcome depends on timing — leading to inconsistent results.
- **How to prevent:** synchronization, locks, atomic variables (`AtomicInteger`), or immutable objects.

**42. `wait()`/`notify()` vs `sleep()`?**
- `sleep()` pauses the current thread without releasing any lock. `wait()` releases the lock and pauses until `notify()`/`notifyAll()` is called — used for inter-thread communication.

**43. What is `ExecutorService`?**
- **What:** A higher-level API for managing a pool of threads instead of manually creating `Thread` objects.
- **Why:** Reuses threads (avoids overhead of creating a new thread per task), manages lifecycle, supports scheduling.
```java
ExecutorService pool = Executors.newFixedThreadPool(4);
pool.submit(() -> processPayroll());
pool.shutdown();
```

**44. What is `volatile`?**
- **What:** Ensures a variable's value is always read from/written to main memory, not a thread's local CPU cache.
- **Why:** Guarantees visibility of changes across threads (though it does NOT guarantee atomicity of compound operations like `i++`).

**45. Deadlock — what and how to avoid?**
- **What:** Two+ threads waiting on each other's locks forever.
- **How to avoid:** Consistent lock ordering across all threads, using timeouts (`tryLock`), minimizing nested locks.

---

## Section 6: JVM & Memory

**46. What is JVM, JRE, JDK?**
- **JVM** — runs bytecode. **JRE** — JVM + libraries needed to *run* Java apps. **JDK** — JRE + compiler/tools needed to *develop* Java apps.

**47. Heap vs Stack memory?**
- **Heap** — stores objects, shared across threads, garbage collected.
- **Stack** — stores method call frames + local variables, per-thread, automatically cleaned up when a method returns.

**48. What is Garbage Collection?**
- **What:** Automatic reclamation of memory occupied by objects no longer reachable.
- **Why:** Removes manual memory management burden (no `free()` like C).
- **How:** Generational GC — most objects die young (Young Gen), survivors get promoted to Old Gen; different collectors (G1, ZGC) tune the tradeoff between pause time and throughput.

**49. What is a Memory Leak in Java (if GC exists)?**
- Objects that are technically *reachable* (e.g., held in a static collection, or an unclosed listener/cache) but never actually used again — GC can't collect them because something still references them.

**50. What is the difference between `==` and `.equals()`?**
- `==` compares references (memory address) for objects, or actual value for primitives. `.equals()` compares logical/content equality (if overridden).
```java
String a = new String("hi");
String b = new String("hi");
a == b;        // false — different objects
a.equals(b);   // true — same content
```

**51. What is String immutability and why?**
- **What:** Once created, a `String`'s value can't change — any "modification" creates a new object.
- **Why:** Enables safe sharing across threads (no sync needed), allows the String pool (interned literals reused), and is required for security (e.g., strings used as class loader/file paths can't be altered mid-use).

**52. String pool — what is it?**
- A special memory area where string literals are cached/reused. `String s = "abc"` reuses an existing "abc" in the pool if present; `new String("abc")` forces a new object on the heap.

---

## Section 7: Miscellaneous / Frequently Asked

**53. What is the difference between `this` and `super`?**
- `this` refers to the current object instance; `super` refers to the immediate parent class.

**54. What are Java access modifiers?**
- `private` (class only), default/package-private (same package), `protected` (package + subclasses), `public` (everywhere).

**55. What is Autoboxing/Unboxing?**
- Automatic conversion between primitives and their wrapper classes (`int` ↔ `Integer`). Can cause subtle bugs/performance issues (e.g., `==` comparing `Integer` objects outside the cached range -128 to 127 gives unexpected `false`).

**56. What is the diamond problem and how does Java avoid it?**
- Ambiguity when a class inherits conflicting methods from two parents (via multiple inheritance). Java avoids this for classes (single inheritance only) but interfaces can cause a version of it with `default` methods — resolved by requiring the implementing class to explicitly override the conflicting method.

---

## Quick tips for your interviews
- For `HashMap` internals, be ready to draw it out — bucket array, hashing, collision handling, treeification at 8 entries, resize at 0.75 load factor. This is asked *very* often at 3–5 YOE level.
- Tie concurrency answers to your notification service rate-limiting work — real experience with thread safety/idempotency lands well.
- For OOP questions, use Humine examples (`Employee`, `LeaveRequest`, `PayrollService`) instead of generic `Animal`/`Shape` examples — it shows real applied understanding.

Want a mock round where I fire these at you one by one and grade your spoken/typed answers?
