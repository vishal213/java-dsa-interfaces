title=Java DSA Reference
date=2024-05-25
type=page
status=published
~~~~~~
# Java DSA Quick Reference

Daily cheat sheet covering 15 foundational Java data structures. Each entry lists common initialization patterns, CRUD snippets, and a real-world use case to help cement when to reach for the structure.

---

## 1. ArrayList
- **Init**
  ```java
  List<String> names = new ArrayList<>();
  ```
- **CRUD**
  ```java
  names.add("Alice");           // Create
  String first = names.get(0);  // Read
  names.set(0, "Bob");          // Update
  names.remove("Bob");          // Delete by value
  ```
- **Use case**: Dynamic arrays for UI dropdown options or maintaining ordered search results where random access is required.

## 2. LinkedList
- **Init**
  ```java
  Deque<Integer> queue = new LinkedList<>();
  ```
- **CRUD**
  ```java
  queue.addLast(10);        // enqueue
  int head = queue.peek();  // peek
  queue.removeFirst();      // dequeue
  queue.clear();            // remove all
  ```
- **Use case**: Implementing LRU caches where frequent head/tail updates occur (e.g., session eviction).

## 3. ArrayDeque
- **Init**
  ```java
  Deque<String> stack = new ArrayDeque<>();
  ```
- **CRUD**
  ```java
  stack.push("a");          // push
  String top = stack.peek();
  stack.pop();              // pop
  ```
- **Use case**: High-performance stack/queue for parsing expressions or BFS without synchronization overhead.

## 4. Stack
- **Init**
  ```java
  Stack<Character> brackets = new Stack<>();
  ```
- **CRUD**
  ```java
  brackets.push('(');
  char current = brackets.peek();
  brackets.pop();
  ```
- **Use case**: Balanced parentheses validation or undo/redo history tracking.

## 5. Queue (LinkedList implementation)
- **Init**
  ```java
  Queue<Message> messages = new LinkedList<>();
  ```
- **CRUD**
  ```java
  messages.offer(new Message("payment"));
  Message next = messages.peek();
  messages.poll(); // returns null if empty
  ```
- **Use case**: Buffering external system events to process at consumer pace when producers are bursty.

## 6. PriorityQueue
- **Init**
  ```java
  PriorityQueue<Integer> minHeap = new PriorityQueue<>();
  ```
- **CRUD**
  ```java
  minHeap.add(5);
  minHeap.offer(1);
  int smallest = minHeap.peek();
  minHeap.poll(); // removes smallest
  ```
- **Use case**: Scheduling jobs by priority or tracking top-K trending items with efficient extraction.

## 7. HashMap
- **Init**
  ```java
  Map<String, Integer> wordCount = new HashMap<>();
  ```
- **CRUD**
  ```java
  wordCount.put("java", 1);
  int count = wordCount.getOrDefault("java", 0);
  wordCount.put("java", count + 1);
  wordCount.remove("java");
  ```
- **Use case**: Counting request frequencies per endpoint for rate limiting or analytics.

## 8. LinkedHashMap
- **Init**
  ```java
  Map<String, Integer> ordered = new LinkedHashMap<>();
  ```
- **CRUD**
  ```java
  ordered.put("a", 1);
  ordered.put("b", 2);
  ordered.get("a");
  ordered.remove("b");
  ```
- **Use case**: LRU cache (override `removeEldestEntry`) or preserving insertion order for request tracing.

## 9. TreeMap
- **Init**
  ```java
  NavigableMap<Integer, String> sorted = new TreeMap<>();
  ```
- **CRUD**
  ```java
  sorted.put(10, "low");
  sorted.put(20, "medium");
  String first = sorted.firstEntry().getValue();
  sorted.remove(10);
  ```
- **Use case**: Range queries such as finding nearest price points or time-based scheduling.

## 10. HashSet
- **Init**
  ```java
  Set<String> seen = new HashSet<>();
  ```
- **CRUD**
  ```java
  seen.add("order-1");
  boolean exists = seen.contains("order-1");
  seen.remove("order-1");
  ```
- **Use case**: Deduplicating user IDs in batch imports or tracking processed messages.

## 11. LinkedHashSet
- **Init**
  ```java
  Set<String> recent = new LinkedHashSet<>();
  ```
- **CRUD**
  ```java
  recent.add("u1");
  recent.add("u2");
  recent.iterator().next(); // oldest inserted
  recent.remove("u1");
  ```
- **Use case**: Maintaining deduplicated, ordered history such as recently viewed products.

## 12. TreeSet
- **Init**
  ```java
  NavigableSet<Integer> scores = new TreeSet<>();
  ```
- **CRUD**
  ```java
  scores.add(85);
  scores.add(92);
  int best = scores.last();
  scores.remove(85);
  ```
- **Use case**: Leaderboards requiring sorted access or finding floor/ceiling values in matching engines.

## 13. ConcurrentHashMap
- **Init**
  ```java
  ConcurrentMap<String, Long> stats = new ConcurrentHashMap<>();
  ```
- **CRUD**
  ```java
  stats.putIfAbsent("hits", 0L);
  stats.compute("hits", (k, v) -> v + 1);
  stats.remove("hits", 0L);
  ```
- **Use case**: Thread-safe counters like tracking active sessions per tenant in multi-threaded services.

## 14. CopyOnWriteArrayList
- **Init**
  ```java
  List<String> subscribers = new CopyOnWriteArrayList<>();
  ```
- **CRUD**
  ```java
  subscribers.add("alice@example.com");
  for (String s : subscribers) {
      // snapshot iteration
  }
  subscribers.remove("alice@example.com");
  ```
- **Use case**: Observer lists for configuration listeners where reads dominate and modifications are rare.

## 15. BlockingQueue (ArrayBlockingQueue example)
- **Init**
  ```java
  BlockingQueue<Order> pending = new ArrayBlockingQueue<>(100);
  ```
- **CRUD**
  ```java
  pending.put(new Order("o1"));      // blocks if full
  Order next = pending.take();       // blocks if empty
  pending.offer(new Order("o2"));    // non-blocking
  ```
- **Use case**: Producer-consumer pipelines such as ingestion workers buffering work before processing threads.

## 16. PriorityBlockingQueue
- **Init**
  ```java
  BlockingQueue<Job> scheduled = new PriorityBlockingQueue<>();
  ```
- **CRUD**
  ```java
  scheduled.put(new Job(1));       // insert with priority
  Job next = scheduled.take();     // always smallest priority first
  scheduled.removeIf(Job::isDone);
  ```
- **Use case**: Background job schedulers prioritizing urgent tasks (e.g., fraud alerts) without starving lower priority work.

## 17. ConcurrentLinkedQueue
- **Init**
  ```java
  Queue<Event> events = new ConcurrentLinkedQueue<>();
  ```
- **CRUD**
  ```java
  events.offer(new Event("login"));
  Event peek = events.peek();
  events.poll();                 // non-blocking removal
  events.removeIf(Event::expired);
  ```
- **Use case**: Lock-free ingestion buffers for metrics/log events in high-throughput microservices.

## 18. EnumSet
- **Init**
  ```java
  EnumSet<Permission> perms = EnumSet.noneOf(Permission.class);
  ```
- **CRUD**
  ```java
  perms.add(Permission.READ);
  boolean allowed = perms.contains(Permission.WRITE);
  perms.remove(Permission.READ);
  perms.addAll(EnumSet.of(Permission.EXPORT, Permission.DELETE));
  ```
- **Use case**: Feature flag combinations or access control checks when domain values are finite enums.

## 19. EnumMap
- **Init**
  ```java
  Map<DayOfWeek, Integer> capacity = new EnumMap<>(DayOfWeek.class);
  ```
- **CRUD**
  ```java
  capacity.put(DayOfWeek.MONDAY, 120);
  int seats = capacity.getOrDefault(DayOfWeek.FRIDAY, 0);
  capacity.replace(DayOfWeek.MONDAY, 150);
  capacity.remove(DayOfWeek.SUNDAY);
  ```
- **Use case**: Scheduling or pricing tables keyed by enum types (e.g., daily staffing levels).

## 20. BitSet
- **Init**
  ```java
  BitSet flags = new BitSet();
  ```
- **CRUD**
  ```java
  flags.set(10);              // mark bit
  boolean on = flags.get(10);
  flags.clear(10);
  flags.flip(0, 64);          // toggle range
  ```
- **Use case**: Tracking feature availability per user ID or managing large boolean matrices (e.g., prime sieve) with compact memory.

---

## How to Practice
- Build tiny REPL-style tests (JUnit or `main`) for each structure and simulate daily tasks (e.g., queue draining, map aggregation).
- Time box 5 minutes daily to review 2-3 structures and recall CRUD + use case without peeking.
- Combine structures (e.g., `PriorityQueue` + `HashMap` for LFU cache) to deepen understanding.
