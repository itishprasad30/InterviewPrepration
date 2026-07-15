# Java Stream API Interview Prep — 30 Questions (What / Why / How + Example)

Reference domain used in examples: `List<Employee> employees` where `Employee` has `name`, `dept`, `salary`, `age`.

---

## Section 1: Fundamentals

**1. What is the Stream API and why was it introduced (Java 8)?**
- **What:** An abstraction for processing sequences of elements declaratively — filter, transform, aggregate — instead of writing manual loops.
- **Why:** Cuts boilerplate, reads closer to *what* you want rather than *how* to loop, and enables easy parallelization without rewriting logic.
```java
// Imperative
List<String> names = new ArrayList<>();
for (Employee e : employees) {
    if (e.getSalary() > 50000) names.add(e.getName());
}
// Stream
List<String> names = employees.stream()
    .filter(e -> e.getSalary() > 50000)
    .map(Employee::getName)
    .collect(Collectors.toList());
```

**2. Is a Stream a data structure?**
- **No.** It doesn't store elements — it's a pipeline that pulls from a source (collection, array, I/O channel) and processes elements on demand.

**3. What are the three parts of a stream pipeline?**
- **Source** (e.g., `list.stream()`), **intermediate operations** (`filter`, `map` — lazy, return a new stream), **terminal operation** (`collect`, `forEach`, `reduce` — triggers actual execution).

**4. Why are intermediate operations "lazy"?**
- **What:** Nothing runs until a terminal operation is invoked.
- **Why:** Allows the JVM to optimize the whole pipeline (e.g., short-circuit early) instead of processing the full list at every step.
```java
employees.stream()
    .filter(e -> { System.out.println("filtering " + e.getName()); return e.getSalary() > 50000; })
    .map(Employee::getName);
// Nothing prints yet — no terminal operation called!
```

**5. Can a Stream be reused/consumed twice?**
- **No.** Once a terminal operation runs, the stream is closed. Reusing it throws `IllegalStateException: stream has already been operated upon or closed`.
```java
Stream<String> s = names.stream();
s.forEach(System.out::println);
s.forEach(System.out::println); // throws exception
```

**6. Stream vs Collection — key difference?**
- Collection is about **storing** data; Stream is about **computing/processing** data. Streams don't mutate the source (unless you explicitly do so inside a lambda, which is discouraged).

---

## Section 2: Core Intermediate Operations

**7. `filter()` — what and example?**
- Keeps only elements matching a `Predicate`.
```java
List<Employee> highEarners = employees.stream()
    .filter(e -> e.getSalary() > 80000)
    .collect(Collectors.toList());
```

**8. `map()` — what and example?**
- Transforms each element 1-to-1 into another form.
```java
List<String> names = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.toList());
```

**9. `map()` vs `flatMap()`?**
- `map()` — one input → one output (even if the output is itself a list, you get a `Stream<List<X>>`).
- `flatMap()` — one input → a stream of outputs, then **flattens** all those streams into a single stream. Needed when mapping produces nested collections.
```java
// departments -> each has a List<Employee>
List<Employee> allEmployees = departments.stream()
    .flatMap(dept -> dept.getEmployees().stream())
    .collect(Collectors.toList());
```

**10. `sorted()` — how do you sort with custom logic?**
```java
List<Employee> sorted = employees.stream()
    .sorted(Comparator.comparing(Employee::getSalary).reversed())
    .collect(Collectors.toList());

// Multi-level sort
employees.stream()
    .sorted(Comparator.comparing(Employee::getDept)
                       .thenComparing(Employee::getSalary, Comparator.reverseOrder()))
    .collect(Collectors.toList());
```

**11. `distinct()` — what does it rely on?**
- Removes duplicates based on `equals()`/`hashCode()` of the elements.

**12. `limit()` and `skip()` — what and use case?**
- `limit(n)` — takes first n elements (useful for pagination/top-N). `skip(n)` — discards first n elements.
```java
// Top 5 highest earners
employees.stream()
    .sorted(Comparator.comparing(Employee::getSalary).reversed())
    .limit(5)
    .collect(Collectors.toList());
```

**13. `peek()` — what is it for (and a common misuse)?**
- **What:** Lets you "peek" at elements as they flow through, mainly for debugging — doesn't transform anything.
- **Misuse warning:** Because streams are lazy, `peek()` **won't execute** without a terminal operation downstream — relying on it for side effects/business logic is a common bug.

---

## Section 3: Terminal Operations

**14. `collect()` — what and common collectors?**
- Terminal op that gathers stream elements into a result container.
```java
List<String> list = stream.collect(Collectors.toList());
Set<String> set = stream.collect(Collectors.toSet());
String joined = stream.collect(Collectors.joining(", "));
```

**15. `Collectors.groupingBy()` — what and example?**
- Groups elements by a classifier function, similar to SQL `GROUP BY`.
```java
Map<String, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDept));

// With downstream aggregation
Map<String, Double> avgSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDept, Collectors.averagingDouble(Employee::getSalary)));
```

**16. `Collectors.partitioningBy()` — how is it different from `groupingBy()`?**
- Splits into exactly **two** groups based on a boolean `Predicate` (`true`/`false` keys), whereas `groupingBy` can produce any number of groups.
```java
Map<Boolean, List<Employee>> partitioned = employees.stream()
    .collect(Collectors.partitioningBy(e -> e.getSalary() > 50000));
```

**17. `reduce()` — what and example?**
- Combines all elements into a single result using an accumulator function.
```java
// Sum of all salaries
double total = employees.stream()
    .map(Employee::getSalary)
    .reduce(0.0, Double::sum);

// Find max without Collectors.maxBy
Optional<Employee> highestPaid = employees.stream()
    .reduce((e1, e2) -> e1.getSalary() > e2.getSalary() ? e1 : e2);
```

**18. `count()`, `anyMatch()`, `allMatch()`, `noneMatch()` — what and example?**
```java
long count = employees.stream().filter(e -> e.getDept().equals("IT")).count();
boolean anyHighEarner = employees.stream().anyMatch(e -> e.getSalary() > 100000);
boolean allAdults = employees.stream().allMatch(e -> e.getAge() >= 18);
boolean noneUnpaid = employees.stream().noneMatch(e -> e.getSalary() == 0);
```
- **Why they matter:** These **short-circuit** — `anyMatch` stops at the first match instead of scanning everything, unlike manually looping with a flag.

**19. `findFirst()` vs `findAny()`?**
- `findFirst()` — deterministic, returns the first element in encounter order (important in ordered/sequential streams).
- `findAny()` — no ordering guarantee; in **parallel streams**, this can be faster since it doesn't need to respect order.

**20. `forEach()` vs `forEachOrdered()`?**
- `forEach()` — no ordering guarantee in parallel streams (for performance). `forEachOrdered()` — preserves encounter order even in parallel streams (at the cost of some parallelism benefit).

---

## Section 4: Optional (tightly related to Streams)

**21. What is `Optional` and why does it matter here?**
- **What:** A container that may or may not hold a value — many stream terminal ops (`findFirst`, `max`, `reduce` without identity) return `Optional` instead of risking `null`.
- **Why:** Forces you to explicitly handle the "no result" case.
```java
Optional<Employee> topEarner = employees.stream()
    .max(Comparator.comparing(Employee::getSalary));

topEarner.ifPresentOrElse(
    e -> System.out.println(e.getName()),
    () -> System.out.println("No employees found")
);

String name = topEarner.map(Employee::getName).orElse("N/A");
```

**22. `orElse()` vs `orElseGet()` — what's the subtle difference?**
- `orElse(x)` — **always evaluates x**, even if the Optional has a value (can be wasteful/costly if `x` is an expensive call).
- `orElseGet(supplier)` — only invokes the supplier if the Optional is empty (lazy).
```java
optional.orElse(expensiveDefault());     // expensiveDefault() runs EVERY time
optional.orElseGet(() -> expensiveDefault()); // only runs if optional is empty
```

---

## Section 5: Parallel Streams

**23. What is a parallel stream and how do you create one?**
- **What:** Splits the workload across multiple threads (using the common `ForkJoinPool`) automatically.
```java
long count = employees.parallelStream().filter(e -> e.getSalary() > 50000).count();
```

**24. When should you NOT use a parallel stream?**
- Small datasets (thread coordination overhead outweighs benefit), operations with shared mutable state (race conditions), I/O-bound operations, or when order matters and you're using unordered-friendly ops incorrectly. Rule of thumb: only helps for large datasets with CPU-bound, stateless operations.

**25. What's a danger of using shared mutable state inside a parallel stream?**
```java
// BAD — race condition, ArrayList isn't thread-safe
List<String> result = new ArrayList<>();
employees.parallelStream().forEach(e -> result.add(e.getName())); // unsafe!

// GOOD — use collect(), which handles thread-safety internally
List<String> result = employees.parallelStream()
    .map(Employee::getName)
    .collect(Collectors.toList());
```

---

## Section 6: Numeric & Primitive Streams

**26. Why do `IntStream`, `LongStream`, `DoubleStream` exist separately from `Stream<T>`?**
- **Why:** Avoids the overhead of autoboxing/unboxing (`Integer` object wrapping) when working purely with primitives — better performance for numeric-heavy operations.
```java
int totalAge = employees.stream().mapToInt(Employee::getAge).sum();
IntSummaryStatistics stats = employees.stream().mapToInt(Employee::getAge).summaryStatistics();
stats.getMax(); stats.getAverage(); stats.getMin();
```

**27. `IntStream.range()` vs `IntStream.rangeClosed()`?**
- `range(1, 5)` → 1,2,3,4 (end exclusive). `rangeClosed(1, 5)` → 1,2,3,4,5 (end inclusive).

---

## Section 7: Common Interview Coding Problems

**28. Find the second highest salary using Streams.**
```java
Optional<Double> secondHighest = employees.stream()
    .map(Employee::getSalary)
    .distinct()
    .sorted(Comparator.reverseOrder())
    .skip(1)
    .findFirst();
```

**29. Group employees by department and get count per department.**
```java
Map<String, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDept, Collectors.counting()));
```

**30. Find duplicate names in a list using Streams.**
```java
Set<String> seen = new HashSet<>();
Set<String> duplicates = employees.stream()
    .map(Employee::getName)
    .filter(name -> !seen.add(name))   // add() returns false if already present
    .collect(Collectors.toSet());
```

**Bonus — Convert a `List<Employee>` into a `Map<String, Double>` of name → salary:**
```java
Map<String, Double> nameToSalary = employees.stream()
    .collect(Collectors.toMap(Employee::getName, Employee::getSalary));
// Watch out: throws IllegalStateException on duplicate keys unless you supply a merge function:
Collectors.toMap(Employee::getName, Employee::getSalary, (a, b) -> a); // keep first on conflict
```

---

## Quick tips for your interviews
- **Q9 (`map` vs `flatMap`) and Q15-16 (`groupingBy`/`partitioningBy`)** are asked almost every single time at 3–5 YOE — know them cold, live-code them without hesitation.
- **Q22 (`orElse` vs `orElseGet`)** is a classic "gotcha" question interviewers use to test attention to detail — mention the eager-vs-lazy evaluation explicitly, it signals depth.
- If asked to solve any of Q28-30 live, narrate your thinking out loud (e.g., "I'll sort descending, dedupe first so ties don't skew skip(1)...") — interviewers weigh reasoning as much as the final code.
- Be ready to explain **why** streams over loops isn't always "better" — for simple single-pass logic, a plain loop can be more readable and avoid the debugging difficulty of lazy pipelines. Shows maturity, not just syntax memorization.

Want this folded into the mixed mock-panel round along with your Spring Boot, Core Java, and SQL docs?
