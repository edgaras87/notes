# What is a Stream? Why it’s useful

A **Stream** is a sequence of elements supporting **functional-style operations** (map, filter, reduce) on collections, arrays, files, etc. It helps you write **concise, readable, parallelizable** code without manual loops and temporary lists.

---

# Quick glossary

* **Source**: Where elements come from (e.g., `List`, array, file).
* **Intermediate op**: Returns a new stream (lazy). Examples: `filter`, `map`, `sorted`.
* **Terminal op**: Triggers processing, produces a value/collection/side-effect. Examples: `collect`, `count`, `forEach`.
* **Lazy**: Nothing runs until a terminal operation is called.
* **Stateless vs stateful**: `map` is stateless; `sorted`/`distinct` are stateful (need to see more than one element).
* **Boxed vs primitive streams**: `Stream<T>` vs `IntStream/LongStream/DoubleStream` (no boxing, extra math ops).
* **Collector**: Strategy to accumulate stream results (e.g., `Collectors.toList()`).
* **Short-circuiting**: Stops early (e.g., `findFirst`, `anyMatch`, `limit`).
* **Ordered**: Some sources are ordered (`List`); some aren’t (`HashSet`).

---

# Creating instances

```java
// From collections & arrays
List<String> names = List.of("Ana","Bob","Anya","Bo");
Stream<String> s1 = names.stream();
Stream<String> s2 = names.parallelStream();
IntStream s3 = Arrays.stream(new int[]{1,2,3});

// Literals / empty
Stream<String> s4 = Stream.of("a","b","c");
Stream<Object> s5 = Stream.empty();

// Ranges / iterate / generate
IntStream s6 = IntStream.range(0, 3);           // 0,1,2
IntStream s7 = IntStream.rangeClosed(1, 3);     // 1,2,3
Stream<Integer> s8 = Stream.iterate(1, n -> n+1).limit(3);    // 1,2,3
Stream<Integer> s9 = Stream.iterate(1, n -> n <= 3, n -> n+1); // Java 9+ bounded
Stream<Double> s10 = Stream.generate(Math::random).limit(2);

// Files, regex, builder
// (wrap in try-with-resources in real code)
try (Stream<String> lines = java.nio.file.Files.lines(java.nio.file.Path.of("data.txt"))) {}
Stream<String> words = java.util.regex.Pattern.compile("\\s+").splitAsStream("hi there");
Stream<String> built = Stream.<String>builder().add("x").add("y").build();
```

**Typical output (examples):**

* `IntStream.range(0,3).boxed().toList()` → `[0, 1, 2]`
* `Stream.of("a","b").toList()` (J16+) → `["a", "b"]`

---

# Reading state / accessors (terminal)

```java
List<Integer> nums = List.of(3,1,4,1,5);
long c = nums.stream().count();                     // -> 5
int min = nums.stream().mapToInt(i->i).min().orElse(-1); // -> 1
int max = nums.stream().mapToInt(i->i).max().orElse(-1); // -> 5
double avg = nums.stream().mapToInt(i->i).average().orElse(0); // -> 2.8
Optional<Integer> first = nums.stream().findFirst(); // -> Optional[3]
```

---

# Checking properties

```java
List<String> l = List.of("a","bb","ccc");
boolean anyLong = l.stream().anyMatch(s -> s.length() >= 3); // -> true
boolean allShort = l.stream().allMatch(s -> s.length() <= 3); // -> true
boolean noneEmpty = l.stream().noneMatch(String::isEmpty);    // -> true
boolean isPar = l.parallelStream().isParallel();              // -> true
```

---

# Transformations (pure ops, no side effects)

```java
List<String> raw = List.of("  a","b  ","  c  ","b");
List<String> cleaned =
  raw.stream()
     .map(String::trim)          // ["a","b","c","b"]
     .filter(s -> !s.isEmpty())  // same
     .distinct()                 // ["a","b","c"] (order preserved)
     .sorted()                   // ["a","b","c"]
     .toList(); // J16+

// flatMap
List<String> phrases = List.of("a b", "c");
List<String> tokens =
  phrases.stream()
         .flatMap(p -> Arrays.stream(p.split("\\s+")))
         .toList(); // -> ["a","b","c"]

// limit / skip
List<Integer> first2 = IntStream.rangeClosed(1,5).limit(2).boxed().toList(); // [1,2]
List<Integer> skip2  = IntStream.rangeClosed(1,5).skip(2).boxed().toList();  // [3,4,5]

// peek (debug only, don’t mutate state!)
List<Integer> out =
  IntStream.range(1,4)
           .peek(i -> System.out.println("saw " + i)) // prints 1,2,3 during terminal op
           .map(i -> i*i)
           .boxed()
           .toList(); // -> [1,4,9]
```

---

# Conversions (to other types)

```java
// To collections
List<String> list1 = Stream.of("a","b").toList();             // J16+ (unmodifiable)
List<String> list2 = Stream.of("a","b").collect(Collectors.toList()); // modifiable (usually)
Set<String> set = Stream.of("a","b","a").collect(Collectors.toSet()); // -> ["a","b"] (order undefined)
Map<Integer, String> map = Stream.of("a","bb","ccc")
  .collect(Collectors.toMap(String::length, s -> s)); // beware duplicate keys

// To array
String[] arr = Stream.of("x","y").toArray(String[]::new);

// To primitives / from primitives
IntStream ints = Stream.of(1,2,3).mapToInt(Integer::intValue);
Stream<Integer> boxed = IntStream.range(1,3).boxed();

// Joining & reducing
String joined = Stream.of("a","b","c").collect(Collectors.joining(",")); // -> "a,b,c"
int sum = IntStream.of(1,2,3).sum();               // -> 6
int prod = IntStream.of(1,2,3).reduce(1, (a,b) -> a*b); // -> 6
```

---

# Iteration & comparison

```java
// Iterate (with side effects) — prefer terminal ops for side effects, not intermediate
List<String> items = List.of("A","B","C");
items.stream().forEach(System.out::println);          // prints in encounter order (or use forEachOrdered)
items.stream().forEachOrdered(System.out::println);   // stable order even in parallel streams

// Compare two sequences (content equality)
boolean same = List.of(1,2,3).stream().toList().equals(List.of(1,2,3)); // -> true
// Or compare after materializing (common & simple):
boolean same2 = Arrays.equals(
  List.of(1,2,3).stream().mapToInt(Integer::intValue).toArray(),
  IntStream.of(1,2,3).toArray()
); // -> true
```

---

# Common utilities (helpers you’ll reach for)

```java
// Comparators
record Person(String name, int age) {}
List<Person> people = List.of(new Person("Ana",30), new Person("Bob",25), new Person("Bo",25));

List<Person> byAgeThenName =
  people.stream()
        .sorted(Comparator.comparingInt(Person::age).thenComparing(Person::name))
        .toList(); // -> [(Bob,25),(Bo,25),(Ana,30)]

// Grouping / partitioning
Map<Integer, List<Person>> byAge =
  people.stream().collect(Collectors.groupingBy(Person::age));
// -> {25=[(Bob,25),(Bo,25)], 30=[(Ana,30)]}

Map<Boolean, List<Person>> partition =
  people.stream().collect(Collectors.partitioningBy(p -> p.age() >= 30));
// -> {true=[(Ana,30)], false=[(Bob,25),(Bo,25)]}

// Summarizing
IntSummaryStatistics stats =
  people.stream().collect(Collectors.summarizingInt(Person::age));
// stats.getCount()=3, getMin()=25, getMax()=30, getAverage()=26.666...
```

---

# Gotchas / anti-patterns / version notes

* **Streams are single-use**: Once a terminal op runs, the stream is consumed. Create a new stream if you need to run another terminal op.
* **Side effects in intermediate ops** (`map`, `filter`, `peek`) are error-prone. Keep them *pure*. Use `peek` only for debugging/logging.
* **`parallelStream()`**: Only beneficial for CPU-heavy, stateless, large workloads. Avoid for small lists, IO-bound work, or when order matters. Be careful with thread-unsafe code.
* **Ordering**: `HashSet.stream()` has no deterministic order; operations like `sorted()` will impose order at a cost.
* **`Collectors.toMap` duplicate keys** throw `IllegalStateException`. Provide a merge function:

  ```java
  var m = Stream.of("a","bb","aa")
    .collect(Collectors.toMap(String::length, s->s, (a,b)->a)); // keep first
  ```
* **`Stream.toList()` (Java 16+)** returns **unmodifiable** list; `Collectors.toList()` usually returns a **mutable** list (implementation-dependent).
* **`Files.lines(...)`** holds file resources—use try-with-resources.
* **Boxing overhead**: Prefer `IntStream/LongStream/DoubleStream` for numeric crunching.
* **Don’t store streams** in fields; build & consume locally.
* **Java 9+ goodies**: `Stream.iterate(seed, hasNext, next)`, `takeWhile`, `dropWhile` (on ordered streams).

  ```java
  var firstSmall = IntStream.of(1,2,3,2,1).takeWhile(i->i<3).boxed().toList(); // [1,2]
  ```

---

# Mini reference table

| Method                        | What it does                     | Example → Output                                                                    |
| ----------------------------- | -------------------------------- | ----------------------------------------------------------------------------------- |
| `filter(p)`                   | Keep elements matching predicate | `List.of(1,2,3).stream().filter(i->i%2==1).toList()` → `[1,3]`                      |
| `map(f)`                      | Transform each element           | `Stream.of("a","bb").map(String::length).toList()` → `[1,2]`                        |
| `flatMap(f)`                  | Flatten nested streams           | `Stream.of("a b","c").flatMap(s->Arrays.stream(s.split(" "))).toList()` → `[a,b,c]` |
| `distinct()`                  | Remove duplicates (keeps first)  | `Stream.of(1,1,2).distinct().toList()` → `[1,2]`                                    |
| `sorted()`                    | Sort using natural order         | `Stream.of(3,1,2).sorted().toList()` → `[1,2,3]`                                    |
| `limit(n)` / `skip(n)`        | Truncate / skip                  | `IntStream.range(1,6).limit(3).boxed().toList()` → `[1,2,3]`                        |
| `anyMatch/allMatch/noneMatch` | Check membership conditions      | `Stream.of("a","bb").anyMatch(s->s.length()==2)` → `true`                           |
| `findFirst/findAny`           | Get an element (Optional)        | `Stream.of(10,20).findFirst()` → `Optional[10]`                                     |
| `reduce(id,acc)`              | Fold elements                    | `IntStream.of(1,2,3).reduce(0,Integer::sum)` → `6`                                  |
| `collect(...)`                | Aggregate via collectors         | `Stream.of("a","b").collect(Collectors.joining(","))` → `"a,b"`                     |
| `mapToInt/boxed`              | Convert between primitive/boxed  | `Stream.of(1,2).mapToInt(i->i).sum()` → `3`                                         |

---

# End-to-end example (typical outputs shown)

**Task:** From a list of people, get the top 2 cities by average age, list residents per city alphabetically, and produce some quick stats.

```java
import java.util.*;
import java.util.stream.*;
import static java.util.stream.Collectors.*;

record Person(String name, String city, int age) {}

public class Demo {
  public static void main(String[] args) {
    List<Person> people = List.of(
      new Person("Ana","Vilnius",30),
      new Person("Bob","Kaunas",25),
      new Person("Bo","Kaunas",27),
      new Person("Anya","Vilnius",34),
      new Person("Cara","Klaipeda",22)
    );

    // Group by city and compute average age
    Map<String, Double> avgAgeByCity =
      people.stream().collect(groupingBy(Person::city, averagingInt(Person::age)));
    System.out.println("avgAgeByCity=" + avgAgeByCity);
    // -> avgAgeByCity={Kaunas=26.0, Klaipeda=22.0, Vilnius=32.0}

    // Top 2 cities by average age (desc)
    List<String> top2Cities =
      avgAgeByCity.entrySet().stream()
        .sorted(Map.Entry.<String,Double>comparingByValue(Comparator.reverseOrder()))
        .limit(2)
        .map(Map.Entry::getKey)
        .toList();
    System.out.println("top2Cities=" + top2Cities);
    // -> top2Cities=[Vilnius, Kaunas]

    // Residents per city alphabetically (for only top2)
    Map<String, List<String>> residents =
      people.stream()
        .filter(p -> top2Cities.contains(p.city()))
        .collect(groupingBy(Person::city,
                            mapping(Person::name,
                                    collectingAndThen(toList(), l -> {
                                      l.sort(Comparator.naturalOrder());
                                      return l;
                                    }))));
    System.out.println("residents=" + residents);
    // -> residents={Kaunas=[Bo, Bob], Vilnius=[Ana, Anya]}

    // Quick stats overall
    IntSummaryStatistics stats = people.stream().collect(summarizingInt(Person::age));
    System.out.println("count=" + stats.getCount()
      + ", min=" + stats.getMin()
      + ", max=" + stats.getMax()
      + ", avg=" + String.format("%.2f", stats.getAverage()));
    // -> count=5, min=22, max=34, avg=27.60
  }
}
```

---

# Bottom line (best practices)

* **Think pipelines**: `source → (map/filter/…)* → terminal`.
* **Keep intermediate ops pure**; put side effects in terminal ops (`forEach`, I/O).
* **Prefer `toList()` (J16+)** for concise unmodifiable results; use `Collectors.toList()` when you need a mutable list.
* **Mind order & cost**: Put cheap filters early; avoid unnecessary `sorted`/`distinct`.
* **Prefer primitive streams** for numeric performance.
* **Avoid `parallelStream()`** unless you’ve measured a win and your operations are thread-safe, stateless, and CPU-bound.
* **Handle `Optional`** properly—use `orElse`, `orElseGet`, `ifPresent`, not `get()` blindly.
* **Don’t over-stream**: A plain `for` loop is fine for very simple, stateful, or performance-critical mutations.

---

```java




```

# 🔑 Wildcards in Stream API

## `? super T` → **Consumer**

Use when the API **feeds stream elements into something**.

* **`forEach(Consumer<? super T>)`**

  ```java
  Stream<Integer> s = Stream.of(1,2,3);
  Consumer<Number> print = n -> System.out.println(n);
  s.forEach(print); // OK because Consumer<? super Integer>
  ```

* **`allMatch(Predicate<? super T>)`**

  ```java
  Stream<String> words = Stream.of("a", "bb");
  Predicate<Object> notNull = Objects::nonNull; // Predicate<Object> is super of String
  boolean all = words.allMatch(notNull);
  ```

* **`sorted(Comparator<? super T>)`**

  ```java
  Stream<String> s = Stream.of("c","a","b");
  Comparator<Object> cmp = (o1,o2) -> o1.toString().compareTo(o2.toString());
  s.sorted(cmp).forEach(System.out::println);
  ```

---

## `? extends R` → **Producer**

Use when the API **returns new elements produced from the stream**.

* **`map(Function<? super T, ? extends R>)`**

  ```java
  Stream<Integer> s = Stream.of(1,2,3);
  Function<Number, String> f = n -> "Num:" + n;
  Stream<String> out = s.map(f); // R is String
  ```

* **`flatMap(Function<? super T, ? extends Stream<? extends R>>)`**

  ```java
  Stream<String> s = Stream.of("a,b","c");
  Stream<String> flat = s.flatMap(str -> Arrays.stream(str.split(",")));
  ```

---

# 📌 Rule of Thumb (PECS)

* **Producer → Extends**: output side (`map`, `flatMap`)
* **Consumer → Super**: input side (`forEach`, `allMatch`, `sorted`)

---

👉 That’s really all you need:

* If the stream **gives elements to** your function = `? super T`.
* If your function **produces new elements** = `? extends R`.

