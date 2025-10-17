# Java Collection Patterns

Practical patterns that surface repeatedly when working with Java data structures. Each pattern lists its purpose, the supporting interfaces or classes, and a short code example you can adapt quickly.

---

## 1. Iterator Pattern
- **When**: Traverse a collection safely, especially when you need to remove elements during iteration.
- **Key types**: `Iterator<T>`, `Collection.iterator()`.
- **Example**:
  ```java
  Iterator<Order> it = orders.iterator();
  while (it.hasNext()) {
      Order order = it.next();
      if (order.isExpired()) {
          it.remove(); // safe removal; avoids ConcurrentModificationException
      }
  }
  ```
- **Notes**: Use `Iterator.remove()` instead of `collection.remove()` inside the loop to stay fail-fast safe.

## 2. Iterable + Enhanced for-loop
- **When**: Expose custom data structures so callers can use for-each loops.
- **Key types**: `Iterable<T>`, `@Override iterator()`.
- **Example**:
  ```java
  public final class RecentUsers implements Iterable<User> {
      private final Deque<User> users = new ArrayDeque<>();

      @Override
      public Iterator<User> iterator() {
          return users.iterator();
      }
  }

  for (User user : recentUsers) {
      notify(user);
  }
  ```
- **Notes**: Implementing `Iterable` unlocks enhanced for-loops and `StreamSupport` for your custom containers.

## 3. ListIterator for Bidirectional Updates
- **When**: Modify or traverse lists in both directions (e.g., editing documents, diff tools).
- **Key types**: `ListIterator<T>`, obtained via `list.listIterator()`.
- **Example**:
  ```java
  ListIterator<String> li = comments.listIterator();
  while (li.hasNext()) {
      String text = li.next();
      if (text.isBlank()) {
          li.remove();
      } else if (text.contains("TODO")) {
          li.set(text.replace("TODO", "DONE"));
      }
  }
  while (li.hasPrevious()) {
      audit(li.previous());
  }
  ```
- **Notes**: `ListIterator` supports `add`, `set`, `remove`, and reverse traversal without resetting the cursor.

## 4. Map Entry Iteration
- **When**: Iterate through maps efficiently, accessing keys and values together.
- **Key types**: `Map.Entry<K, V>`, `entrySet()`, `keySet()`, `values()`.
- **Example**:
  ```java
  for (Map.Entry<String, Integer> entry : inventory.entrySet()) {
      if (entry.getValue() == 0) {
          restock(entry.getKey());
      }
  }
  ```
- **Notes**: Prefer `entrySet()` when you need both key and value; avoid repeated `map.get(key)` calls inside loops.

## 5. Comparator & Comparable Strategy
- **When**: Order elements consistently across collections and algorithms.
- **Key types**: `Comparable<T>`, `Comparator<T>`, `Collections.sort`, `PriorityQueue`.
- **Example**:
  ```java
  record Invoice(String id, BigDecimal amount) implements Comparable<Invoice> {
      @Override
      public int compareTo(Invoice other) {
          return amount.compareTo(other.amount());
      }
  }

  List<Invoice> invoices = new ArrayList<>(unpaid);
  invoices.sort(Comparator.comparing(Invoice::amount).reversed());
  ```
- **Notes**: Implement `Comparable` for natural ordering; use dedicated `Comparator`s for context-specific sorts.

## 6. Spliterator & Parallel Streams
- **When**: Process large datasets in parallel or build custom stream sources.
- **Key types**: `Spliterator<T>`, `Spliterators`, `StreamSupport`.
- **Example**:
  ```java
  Spliterator<Customer> spliterator = customers.spliterator();
  Stream<Customer> parallelStream = StreamSupport.stream(spliterator, true);
  BigDecimal total = parallelStream
          .map(Customer::balance)
          .reduce(BigDecimal.ZERO, BigDecimal::add);
  ```
- **Notes**: Provide accurate characteristics (`ORDERED`, `SIZED`) for better splitting; use when built-in collections are insufficient.

## 7. Enumeration Bridge (Legacy)
- **When**: Interoperate with older APIs (e.g., `Hashtable`, `Vector`) still returning `Enumeration`.
- **Key types**: `Enumeration<T>`, `Collections.enumeration`, `Collections.list`.
- **Example**:
  ```java
  Enumeration<URL> urls = classLoader.getResources("config.properties");
  List<URL> urlList = Collections.list(urls); // converts to List for modern handling
  ```
- **Notes**: Convert ASAP to modern collections; only use `Enumeration` when interacting with legacy interfaces.

## 8. Collections Utility Wrappers
- **When**: Decorate collections for thread safety, immutability, or read-only views.
- **Key types**: `Collections.unmodifiableList`, `Collections.synchronizedMap`, `Collections.emptyList`.
- **Example**:
  ```java
  List<String> internal = new ArrayList<>();
  List<String> view = Collections.unmodifiableList(internal);

  Map<String, String> shared = Collections.synchronizedMap(new HashMap<>());
  ```
- **Notes**: Unmodifiable wrappers throw `UnsupportedOperationException` on mutation; synchronized wrappers still require client-side iteration synchronization.

## 9. Stream Collector Pattern
- **When**: Derive aggregated structures from streams (`List`, `Map`, groupings).
- **Key types**: `Stream<T>`, `Collectors`, `toList`, `groupingBy`, `joining`.
- **Example**:
  ```java
  Map<Status, List<Order>> byStatus = orders.stream()
          .collect(Collectors.groupingBy(Order::status));

  String csv = orders.stream()
          .map(Order::id)
          .collect(Collectors.joining(","));
  ```
- **Notes**: Collectors compose (`collectingAndThen`, `mapping`) for advanced aggregations. Use `toList()` (Java 16+) for unmodifiable results by default.

## 10. Optional Iteration Helpers
- **When**: Integrate optional values into collection pipelines without verbose null checks.
- **Key types**: `Optional`, `stream()`, `ifPresent`.
- **Example**:
  ```java
  Optional<User> maybeUser = userRepository.findById(id);
  maybeUser.ifPresent(activeUsers::add);

  List<String> emails = maybeUser.stream()
          .map(User::email)
          .toList();
  ```
- **Notes**: `Optional.stream()` (Java 9+) allows seamless inclusion in stream pipelines; `ifPresentOrElse` helps with branching logic.

---

## Practice Tips
- Translate existing loops to iterators or streams to see which pattern fits best.
- When designing custom collections, implement both `Iterable` and useful `Spliterator` characteristics for maximum interoperability.
- Review fail-fast behavior: modifying collections outside iterator patterns triggers exceptions—plan your traversal strategy accordingly.
