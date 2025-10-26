title=Java Developer Interview Prep
date=2024-05-25
type=page
status=published
~~~~~~
# Java Developer Interview Prep

## Table of Contents
- [Core Java / Backend](#core-java--backend)
  - [HashMap vs ConcurrentHashMap](#hashmap-vs-concurrenthashmap)
  - [equals()/hashCode() Contract](#equalscontract)
  - [Immutability Patterns](#immutability-patterns)
  - [Threads vs Executors](#threads-vs-executors)
  - [Java Memory Model Basics](#java-memory-model-basics)
  - [GC Tuning: G1 and ZGC](#gc-tuning-g1-and-zgc)
  - [Java 8, 11, 17 Features in Practice](#java-8-11-17-features-in-practice)
  - [Resilience in Production](#resilience-in-production)
- [DSA / Coding](#dsa--coding)
  - [Arrays and Strings](#arrays-and-strings)
  - [Graphs and Trees](#graphs-and-trees)
  - [Dynamic Programming Staples](#dynamic-programming-staples)
  - [Binary Search Patterns and Stream Pipelines](#binary-search-patterns-and-stream-pipelines)
- [SQL / Database](#sql--database)
  - [Index Design, Deduplication, and Window Functions](#index-design-deduplication-and-window-functions)
  - [Transactions, Pagination, and Read vs Write Models](#transactions-pagination-and-read-vs-write-models)
- [Spring / Microservices](#spring--microservices)
  - [Spring Boot Auto-Configuration and Observability Basics](#spring-boot-auto-configuration-and-observability-basics)
  - [REST Design Essentials](#rest-design-essentials)
  - [Messaging Patterns with Kafka and RabbitMQ](#messaging-patterns-with-kafka-and-rabbitmq)
  - [Observability at Scale](#observability-at-scale)
- [System Design](#system-design)
  - [UPI-Style Payment Service](#upi-style-payment-service)
  - [Transaction Feed Service (TStore Inspired)](#transaction-feed-service-tstore-inspired)
  - [In-App Inbox and Alerts (Bullhorn-Style)](#in-app-inbox-and-alerts-bullhorn-style)
  - [Job Scheduler at Scale (Clockwork-Like)](#job-scheduler-at-scale-clockwork-like)
  - [Metrics Platform for SLOs](#metrics-platform-for-slos)

---

## Core Java / Backend

### HashMap vs ConcurrentHashMap
- **HashMap** is not thread-safe. In a multi-threaded context, concurrent modifications can corrupt the bucket structure, leading to infinite loops or lost updates. It allows one `null` key and multiple `null` values.
- **ConcurrentHashMap** provides thread-safe operations without global locking. It forbids `null` keys or values to avoid ambiguity in `get()` operations under race conditions. Modern implementations (Java 8+) use lock-striping and tree bins to reduce contention and avoid worst-case O(n) lookups.
- Iterators: `HashMap` iterators fail-fast (throw `ConcurrentModificationException`). ConcurrentHashMap iterators are weakly consistent: they reflect the state at or after construction but never throw due to concurrent writes.
- Performance: ConcurrentHashMap uses segmented locking internally. Reads are lock-free while writes synchronize on individual bins. Prefer `ConcurrentHashMap` when frequent concurrent reads and occasional writes happen, but use higher-level abstractions (`computeIfAbsent`, `merge`) to atomically update values and avoid race conditions.

### equals()/_hashCode() Contract
- Contract summary:
  1. If `a.equals(b)` is true, `a.hashCode() == b.hashCode()` must hold.
  2. Consistency: Repeated calls during object lifetime must return the same value if fields used are unchanged.
  3. Symmetry, reflexivity, transitivity must be preserved.
- Violations produce undefined behavior in hash-based collections (`HashSet`, `HashMap`). For example, forgetting to include a field in both `equals` and `hashCode` can allow duplicates or break retrieval.
- Implementation tips:
  - Use `Objects.equals` and `Objects.hash` for null-safe comparisons.
  - Keep `equals` final unless you fully understand inheritance impacts.
  - For performance-critical classes, precompute hash codes for immutable objects to avoid repeated calculations.

### Immutability Patterns
- Immutable objects guarantee thread safety without synchronization because state is fixed after construction.
- Guidelines:
  - Make fields `private final`.
  - No setters; only constructors or builders should populate data.
  - Defensive copies for mutable inputs/outputs.
- Common patterns: `record` classes (Java 16+), `Collections.unmodifiableList`, Guava `ImmutableList`.
- Benefits: safe sharing across threads, easier reasoning, fail-fast design. Trade-offs include potential higher allocation cost and complex object graphs requiring builders.
- Example:

```java
public final class Money {
    private final BigDecimal amount;
    private final Currency currency;

    public Money(BigDecimal amount, Currency currency) {
        this.amount = amount.stripTrailingZeros();
        this.currency = currency;
    }

    public BigDecimal amount() {
        return amount;
    }

    public Currency currency() {
        return currency;
    }
}
```

### Threads vs Executors
- Creating raw `Thread` instances is flexible but leads to brittle lifecycle management. Each `Thread` owns a stack and OS resources; frequent creation hurts performance.
- Executors decouple task submission from execution:
  - `ExecutorService` (fixed, cached, scheduled pools).
  - `ForkJoinPool` for divide-and-conquer tasks.
- Use `Executors.newFixedThreadPool` for predictable workloads, `newCachedThreadPool` for bursty but short-lived tasks (beware unbounded growth). Prefer `ThreadPoolExecutor` constructor for custom policies (queue size, rejection handler, thread factory).
- Graceful shutdown: `shutdown()` to reject new tasks, `awaitTermination()` to block; `shutdownNow()` as best-effort interrupt. Always handle interruptions using `Thread.currentThread().isInterrupted()` checks and propagate `InterruptedException`.
- Example implementations:

```java
public void launchWithThreads(List<Runnable> tasks) {
    List<Thread> threads = new ArrayList<>();
    for (Runnable task : tasks) {
        Thread worker = new Thread(task);
        worker.start();
        threads.add(worker);
    }
    for (Thread worker : threads) {
        try {
            worker.join();
        } catch (InterruptedException ie) {
            Thread.currentThread().interrupt(); // preserve interrupt status for callers
            return;
        }
    }
}
```

- Raw threads give maximum control but force manual lifecycle management and error handling.

```java
public void launchWithExecutor(List<Runnable> tasks) {
    ExecutorService pool = Executors.newFixedThreadPool(
            Runtime.getRuntime().availableProcessors());
    try {
        for (Runnable task : tasks) {
            pool.submit(task);
        }
    } finally {
        pool.shutdown();
    }
    try {
        if (!pool.awaitTermination(30, TimeUnit.SECONDS)) {
            pool.shutdownNow(); // interrupt lingering tasks
        }
    } catch (InterruptedException ie) {
        pool.shutdownNow();
        Thread.currentThread().interrupt();
    }
}
```

- Executors simplify pooling, queueing, and shutdown logic while letting you plug in custom rejection policies and thread factories when needed.

### Java Memory Model Basics
- The JMM defines happens-before relationships, ensuring visibility and ordering guarantees.
- Key tools:
  - `volatile` establishes write-read happens-before but does not ensure atomic compound operations. Use for state flags, not counters.
  - `synchronized` blocks establish mutual exclusion and happens-before between unlock/lock.
  - `java.util.concurrent` classes provide higher-level fences (`AtomicInteger`, `Lock`).
- Safe publication patterns: static initializers, volatile references, final fields, thread-confined objects, and `CompletableFuture`.
- Common pitfalls: double-checked locking requires volatile reference, `stop()` is unsafe because it violates invariants.

### GC Tuning: G1 and ZGC
- **G1 (Garbage-First) Collector**
  - Region-based; prioritizes regions with most garbage first.
  - Targets predictable pause times via `-XX:MaxGCPauseMillis`.
  - Useful for large heaps (multi-GB) with moderate latency requirements.
  - Tuning knobs: adjust concurrent thread counts, enable string dedup (`-XX:+UseStringDeduplication`), monitor with JVM GC logs (`-Xlog:gc*`).
- **ZGC (Z Garbage Collector)**
  - Concurrent, region-based, uses colored pointers and load barriers to keep pauses under 10 ms even on huge heaps (multi-terabytes).
  - Ideal for low-latency services; requires recent JVM (11+ with updates, stable in 15+).
  - Minimal tuning: set reasonable heap ranges (`-Xms`, `-Xmx`), monitor using JFR (Java Flight Recorder).
- GC tuning process:
  1. Gather baseline metrics (allocation rate, pause times).
  2. Identify longest pauses and memory pressure.
  3. Adjust pause goals or heap size; consider enabling GC logging with `-Xlog:gc*:file=gc.log`.
  4. Use tools like GC Easy, JMC for analysis.

### Java 8, 11, 17 Features in Practice
- **Streams (Java 8)**: Express data transformations declaratively; lazy evaluation combined with terminal operations. Prefer parallel streams only when data is large, operations stateless, and splitting efficient.

```java
List<String> topCustomers = orders.stream()
        .filter(order -> order.total() > 1000)
        .sorted(Comparator.comparing(Order::total).reversed())
        .limit(10)
        .map(Order::customerId)
        .toList();
```

- Quick patterns:

```java
List<String> upperCase = names.stream()
        .map(String::toUpperCase)
        .sorted()
        .toList();
```

```java
Map<String, Long> countByCity = users.stream()
        .collect(Collectors.groupingBy(User::city, Collectors.counting()));
```

```java
double averageAge = users.stream()
        .mapToInt(User::age)
        .average()
        .orElse(0);
```

- **Optional**: Fosters null-safe APIs, but avoid replacing all fields with `Optional`. Use as return type.
- **Functional interfaces and method references**: Encourage succinct callbacks, custom collectors.
- **Java 9-11**:
  - `var` (Java 10) reduces verbosity while preserving static typing; use when the initializer reveals type clearly.
  - HTTP Client (Java 11) for reactive HTTP calls, supports HTTP/2 and WebSockets.
  - Local variable type inference in lambdas (Java 11) for annotations on parameters.
- **Java 17**:
  - `record` classes provide immutable data carriers with generated accessors.
  - Sealed classes restrict inheritance hierarchies, improving pattern matching safety.
  - Enhanced `switch` expressions allow concise branching with exhaustiveness checks.
  - Text blocks for multiline strings with clear indentation control.

```java
record Customer(String id, String name) {}

String payload = """
        {
          "type": "PING",
          "timestamp": %d
        }
        """.formatted(System.currentTimeMillis());
```

### Resilience in Production
- **Retries**: Useful for transient failures (timeouts, 5xx). Combine with exponential backoff and jitter to avoid thundering herds. Configure retry count per downstream dependency; avoid retrying non-idempotent operations unless supported.
- **Timeouts**: Every remote call requires a timeout shorter than client timeout to prevent thread exhaustion. Use `CompletableFuture.orTimeout` or `HttpClient` `Duration`.
- **Circuit breakers**: Trip when error rate exceeds threshold, fast-failing to allow downstream recovery (Resilience4j, Spring Cloud Circuit Breaker). Combine with fallback responses or cached data.
- Where applied:
  - Client SDKs invoking downstream services.
  - Data access layers hitting DBs with overloaded queries.
  - External payment gateways with intermittent issues.
- Observability: track retry counts, circuit states, timeout metrics. Document SLA/SLO targets to align with resilience thresholds.

---

## DSA / Coding

### Arrays and Strings
Key interview tasks stress time complexity, space usage, and trade-offs.

#### Two-Sum Variants
- Unsorted array with unique solution:

```java
public int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> indexByValue = new HashMap<>();
    for (int i = 0; i < nums.length; i++) {
        int complement = target - nums[i];
        if (indexByValue.containsKey(complement)) {
            return new int[]{indexByValue.get(complement), i};
        }
        indexByValue.put(nums[i], i);
    }
    throw new IllegalArgumentException("No solution found");
}
```

- Sorted array variant uses two pointers to stay O(n):

```java
public int[] twoSumSorted(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    while (left < right) {
        int sum = nums[left] + nums[right];
        if (sum == target) {
            return new int[]{left, right};
        } else if (sum < target) {
            left++;
        } else {
            right--;
        }
    }
    throw new IllegalArgumentException("No solution found");
}
```

#### Move Zeroes
- Stable in-place approach with two pointers:

```java
public void moveZeroes(int[] nums) {
    int insert = 0;
    for (int num : nums) {
        if (num != 0) {
            nums[insert++] = num;
        }
    }
    while (insert < nums.length) {
        nums[insert++] = 0;
    }
}
```

#### Valid Anagram
- Counting array for lowercase letters; for Unicode prefer `Map<Character, Integer>`.

```java
public boolean isAnagram(String s, String t) {
    if (s.length() != t.length()) {
        return false;
    }
    int[] counts = new int[26];
    for (int i = 0; i < s.length(); i++) {
        counts[s.charAt(i) - 'a']++;
        counts[t.charAt(i) - 'a']--;
    }
    for (int count : counts) {
        if (count != 0) {
            return false;
        }
    }
    return true;
}
```

#### Longest Common Prefix
- Horizontal scanning avoids sorting.

```java
public String longestCommonPrefix(String[] strs) {
    if (strs.length == 0) {
        return "";
    }
    String prefix = strs[0];
    for (int i = 1; i < strs.length; i++) {
        while (!strs[i].startsWith(prefix)) {
            prefix = prefix.substring(0, prefix.length() - 1);
            if (prefix.isEmpty()) {
                return "";
            }
        }
    }
    return prefix;
}
```

### Graphs and Trees
- Choose adjacency lists for sparse graphs. Use iterative traversals when recursion depth may explode.

#### BFS and DFS Templates

```java
public List<Integer> bfs(Map<Integer, List<Integer>> graph, int start) {
    List<Integer> visitedOrder = new ArrayList<>();
    Queue<Integer> queue = new ArrayDeque<>();
    Set<Integer> visited = new HashSet<>();
    queue.add(start);
    visited.add(start);
    while (!queue.isEmpty()) {
        int node = queue.poll();
        visitedOrder.add(node);
        for (int neighbor : graph.getOrDefault(node, List.of())) {
            if (visited.add(neighbor)) {
                queue.add(neighbor);
            }
        }
    }
    return visitedOrder;
}

public List<Integer> dfs(Map<Integer, List<Integer>> graph, int start) {
    List<Integer> order = new ArrayList<>();
    Deque<Integer> stack = new ArrayDeque<>();
    Set<Integer> visited = new HashSet<>();
    stack.push(start);
    while (!stack.isEmpty()) {
        int node = stack.pop();
        if (!visited.add(node)) {
            continue;
        }
        order.add(node);
        List<Integer> neighbors = graph.getOrDefault(node, List.of());
        for (int i = neighbors.size() - 1; i >= 0; i--) {
            stack.push(neighbors.get(i)); // push in reverse to mimic recursive DFS order
        }
    }
    return order;
}
```

#### Shortest Path (Dijkstra)

```java
public Map<Integer, Integer> dijkstra(Map<Integer, List<int[]>> graph, int source) {
    Map<Integer, Integer> dist = new HashMap<>();
    PriorityQueue<int[]> pq = new PriorityQueue<>(Comparator.comparingInt(edge -> edge[1]));
    pq.add(new int[]{source, 0});
    dist.put(source, 0);

    while (!pq.isEmpty()) {
        int[] current = pq.poll();
        int node = current[0];
        int currentDist = current[1];
        if (currentDist > dist.get(node)) {
            continue; // outdated entry
        }
        for (int[] edge : graph.getOrDefault(node, List.of())) {
            int neighbor = edge[0];
            int weight = edge[1];
            int newDist = currentDist + weight;
            if (newDist < dist.getOrDefault(neighbor, Integer.MAX_VALUE)) {
                dist.put(neighbor, newDist);
                pq.add(new int[]{neighbor, newDist});
            }
        }
    }
    return dist;
}
```

#### Tree Re-rooting Technique
- Useful for computing distance sums from all nodes. Perform two DFS passes: first to compute subtree metrics, second to propagate results.

```java
public int[] sumOfDistances(int n, List<List<Integer>> tree) {
    int[] subtreeSize = new int[n];
    int[] answer = new int[n];
    Arrays.fill(subtreeSize, 1);

    dfsPost(0, -1, tree, subtreeSize, answer);
    dfsPre(0, -1, tree, subtreeSize, answer, n);
    return answer;
}

private void dfsPost(int node, int parent, List<List<Integer>> tree, int[] size, int[] answer) {
    for (int neighbor : tree.get(node)) {
        if (neighbor == parent) {
            continue;
        }
        dfsPost(neighbor, node, tree, size, answer);
        size[node] += size[neighbor];
        answer[node] += answer[neighbor] + size[neighbor];
    }
}

private void dfsPre(int node, int parent, List<List<Integer>> tree, int[] size, int[] answer, int n) {
    for (int neighbor : tree.get(node)) {
        if (neighbor == parent) {
            continue;
        }
        answer[neighbor] = answer[node] - size[neighbor] + (n - size[neighbor]);
        dfsPre(neighbor, node, tree, size, answer, n);
    }
}
```

#### Disjoint Set Union (Union-Find)

```java
public static final class DisjointSetUnion {
    private final int[] parent;
    private final int[] rank;

    public DisjointSetUnion(int size) {
        parent = new int[size];
        rank = new int[size];
        for (int i = 0; i < size; i++) {
            parent[i] = i;
        }
    }

    public int find(int x) {
        if (parent[x] != x) {
            parent[x] = find(parent[x]); // path compression
        }
        return parent[x];
    }

    public boolean union(int a, int b) {
        int rootA = find(a);
        int rootB = find(b);
        if (rootA == rootB) {
            return false;
        }
        if (rank[rootA] < rank[rootB]) {
            parent[rootA] = rootB;
        } else if (rank[rootA] > rank[rootB]) {
            parent[rootB] = rootA;
        } else {
            parent[rootB] = rootA;
            rank[rootA]++;
        }
        return true;
    }
}
```

### Dynamic Programming Staples

#### Longest Increasing Subsequence (O(n log n))

```java
public int lengthOfLIS(int[] nums) {
    List<Integer> tails = new ArrayList<>();
    for (int num : nums) {
        int index = Collections.binarySearch(tails, num);
        if (index < 0) {
            index = -index - 1;
        }
        if (index == tails.size()) {
            tails.add(num);
        } else {
            tails.set(index, num);
        }
    }
    return tails.size();
}
```

- `tails[i]` keeps the minimum possible tail of an increasing subsequence of length `i + 1`. Replacing values maintains optimal substructure.

#### Coin Change (Minimum Coins)

```java
public int coinChange(int[] coins, int amount) {
    int max = amount + 1;
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, max);
    dp[0] = 0;

    for (int coin : coins) {
        for (int current = coin; current <= amount; current++) {
            dp[current] = Math.min(dp[current], dp[current - coin] + 1);
        }
    }
    return dp[amount] > amount ? -1 : dp[amount];
}
```

- Iterating coins outermost ensures unlimited usage. Using `max` sentinel avoids overflow.

### Binary Search Patterns and Stream Pipelines

#### Binary Search Templates
- Classic search:

```java
public int binarySearch(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] == target) {
            return mid;
        } else if (nums[mid] < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    return -1;
}
```

- First position greater or equal (lower bound):

```java
public int lowerBound(int[] nums, int target) {
    int left = 0, right = nums.length;
    while (left < right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] < target) {
            left = mid + 1;
        } else {
            right = mid;
        }
    }
    return left;
}
```

- Pattern usage: searching answer space (e.g., minimizing max pages, rope cutting). Formulate monotonic predicate `P(x)` and find boundary where predicate transitions.

#### Stream Transformations
- Pipeline: `filter` narrows dataset, `map` transforms, `reduce` aggregates. Ensure stateless operations for deterministic results.

```java
double totalHighValue = transactions.stream()
        .filter(tx -> tx.status() == Status.SUCCESS)
        .filter(tx -> tx.amount().compareTo(BigDecimal.valueOf(500)) > 0)
        .map(Transaction::amount)
        .reduce(BigDecimal.ZERO, BigDecimal::add)
        .doubleValue();
```

- Use `Collectors.groupingBy` and `Collectors.mapping` for multi-level aggregates. For large data consider `StreamSupport` with Spliterators or switch to parallel streams when operations are CPU-bound and thread-safe.

---

## SQL / Database

### Index Design, Deduplication, and Window Functions
- **Index design**:
  - B-tree indexes speed up range queries and equality filters. Composite index order matters: `(country, created_at)` serves queries filtering by `country` and ordering by `created_at`, but not the reverse unless leftmost prefix is used.
  - Trade-offs: extra write cost (index maintenance), storage overhead, potential slower bulk inserts. Use covering indexes to avoid table lookups.
- **Deduplicating rows**:

```sql
DELETE FROM payments p
USING (
    SELECT id, ROW_NUMBER() OVER (PARTITION BY external_id ORDER BY created_at DESC) AS rnum
    FROM payments
) dup
WHERE p.id = dup.id
  AND dup.rnum > 1;
```

- Alternative: create unique constraints with `ON CONFLICT DO NOTHING` (PostgreSQL) to prevent duplicates.
- **Window functions vs GROUP BY**:
  - `GROUP BY` collapses rows; window functions retain detail rows while computing aggregates.
  - Example: running total and rank simultaneously.

```sql
SELECT user_id,
       created_at,
       amount,
       SUM(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS running_total,
       RANK() OVER (PARTITION BY user_id ORDER BY amount DESC) AS amount_rank
FROM payouts;
```

### Transactions, Pagination, and Read vs Write Models
- **Transaction isolation**:
  - Read uncommitted (rare), read committed (default), repeatable read, serializable. Understand anomalies: dirty reads, non-repeatable reads, phantom reads.
  - Postgres `repeatable read` prevents non-repeatable and phantom reads via MVCC snapshots; MySQL InnoDB's repeatable read uses next-key locks to avoid phantoms.
  - Choose isolation per use case: payments require repeatable read+ with retry logic; analytics can afford read committed.
- **Pagination strategies**:
  - Offset-based: `LIMIT 50 OFFSET 5000` simple but slow for large offsets and can skip rows if inserts happen.
  - Keyset pagination: use last seen key (`WHERE created_at < ? ORDER BY created_at DESC LIMIT 50`) for stable, fast navigation.
  - Seek pagination works only on indexed, monotonically increasing columns.
- **Read vs write models**:
  - CQRS (Command Query Responsibility Segregation) splits write-optimized stores (normalized relational) from read models (denormalized caches, Elasticsearch).
  - Event sourcing feeds read models asynchronously, enabling timeline reconstruction and audit.
  - Feed services often combine write-through cache (Redis) for latest data, plus background jobs to backfill asynchronous projections.

---

## Spring / Microservices

### Spring Boot Auto-Configuration and Observability Basics
- Auto-configuration inspects classpath and configuration properties to create beans. Use `@SpringBootApplication` to trigger `@EnableAutoConfiguration`. Override defaults with `@ConditionalOnBean`, custom `@Configuration`.
- Configuration profiles (`application-dev.yml`, `application-prod.yml`) enable environment-specific properties. Activate via `SPRING_PROFILES_ACTIVE` or command line `--spring.profiles.active=prod`.
- Spring Boot Actuator provides endpoints like `/actuator/health`, `/actuator/metrics`. Secure them using Spring Security or network policies. Customize health indicators by implementing `HealthIndicator`.

### REST Design Essentials
- **Validation**: Use `@Valid` with Bean Validation annotations (`@NotNull`, `@Pattern`). Provide constraint violation handlers to map to consistent error responses.

```java
@PostMapping("/payments")
public ResponseEntity<PaymentResponse> createPayment(@Valid @RequestBody PaymentRequest request) {
    Payment payment = paymentService.create(request);
    return ResponseEntity.status(HttpStatus.CREATED).body(PaymentResponse.from(payment));
}
```

- **Versioning**: URI versioning (`/v1/payments`), header based (`Accept: application/vnd.company.v1+json`), or query param. Maintain backward compatibility; deprecate with documentation and sunset headers.
- **Idempotency**: Use idempotency keys (store request hash). For PUT/PATCH ensure operations update resources deterministically. When retrying POST, check if prior request succeeded before re-executing.
- **Distributed workflows**:
  - SAGA pattern coordinates distributed transactions via orchestrators (central controller) or choreography (events). Outbox pattern ensures messages are stored atomically with DB changes and later published safely.
  - Implement outbox using persistent tables + change data capture (Debezium) or transactional message brokers (Kafka with transaction support).

### Messaging Patterns with Kafka and RabbitMQ
- **Kafka**:
  - Strong ordering within a partition. Select partition key carefully (e.g., payment ID) to guarantee order. Use idempotent producers and transactions to avoid duplicates.
  - Retries: configure `retries`, `acks=all`, and `enable.idempotence=true`. For consumers, handle poison messages with Dead Letter Topics (DLTs).
  - Consumer lag monitoring (`__consumer_offsets`) is essential. For payments, combine with exactly-once semantics using transactional IDs to write to DB and Kafka atomically.
- **RabbitMQ**:
  - Work queues distribute tasks (round-robin). Acknowledge messages after processing; use `prefetch` (QoS) to control in-flight count.
  - Dead-letter exchanges capture failed messages. Use delayed exchanges for retry with backoff.
  - When to choose:
    - Kafka for high-throughput event streaming, log aggregation, ordered events.
    - RabbitMQ for low-latency, complex routing (topic, fanout), request-response patterns.
- Payments flow example:
  1. API publishes to Kafka topic `payment_initiated` with key = payment ID.
  2. Orchestrator service consumes, performs risk checks, writes to DB.
  3. Succeeded events published to settlement topic.
  4. RabbitMQ queue handles synchronous notifications (SMS/email) because of per-message TTL and delayed retries.

### Observability at Scale
- Metrics: instrument via Micrometer (`@Timed`, `Counter`). Export to Prometheus/StatsD.
- Tracing: OpenTelemetry instrumentation with Spring Cloud Sleuth (deprecated) or Micrometer Tracing. Propagate trace IDs via headers (`traceparent`).
- Logs: structured JSON logs with `traceId`, `spanId`. Ship to ELK, Loki.
- Scaling to 300B+ metrics/day:
  - Cardinality control: avoid unbounded label values; use exemplars selectively.
  - Metric aggregation tiers: edge agents perform pre-aggregation before shipping.
  - Sampling and adaptive aggregation for tracing and spans (tail-based sampling).
  - Storage: choose TSDB (Cortex, Mimir, Thanos) with object storage backend. Index retention policies and downsampling for cost control.
  - Alerting: define SLOs using multi-window, multi-burn rate alerting to balance sensitivity and noise.

---

## System Design

### UPI-Style Payment Service
- **Requirements**: instant bank-to-bank transfers, high availability, real-time fraud checks, regulatory compliance (audit logs, reconciliation). Involves participants: payer app, PSP, NPCI-style switch, issuing bank, acquiring bank.
- **Architecture**:
  - API gateway → Auth service (device binding, OAuth) → Payment orchestrator.
  - Orchestrator leverages microservices: risk, limits, ledger, notification.
  - Message queues (Kafka) coordinate state transitions (`INITIATED`, `DEBIT_PENDING`, `CREDIT_PENDING`, `SETTLED`).
  - Use idempotency tokens per transaction; store in Redis with TTL to deduplicate.
  - Ledger service uses append-only datastore (CockroachDB, PostgreSQL with logical replication) ensuring atomic double-entry (debit payer, credit intermediary) with eventual settlement to acquiring bank.
- **Consistency**:
  - Two-phase commit is expensive; instead rely on eventual consistency with compensating transactions (reverse ledger entry).
  - Reconciliation jobs compare transaction logs with bank statements; mismatches escalate manual workflows.
- **Scalability**:
  - Partition ledger by user ID or VPA to reduce contention.
  - Employ read replicas for balance queries; keep writes on primary with synchronous replication to meet RPO.
  - Cache frequent metadata (VPA-to-account mapping) using Redis with invalidation on updates.
- **Security & Compliance**:
  - Encrypt sensitive data at rest (HSM for keys). Use tokenization for account numbers.
  - Implement rate limiting at gateway and device binding to curb fraud.
  - Provide audit trails by persisting every state change with timestamps, actor info.

### Transaction Feed Service (TStore Inspired)
- **Goal**: deliver real-time feed of transactions with pagination, filters, and backfill.
- **Write path**:
  - Kafka topic `transaction_events` ingested by feed builder service.
  - Denormalize into document store (Elasticsearch) optimized for search by user, status, time.
  - Maintain Redis sorted sets for latest N entries per user for fast dashboard loads.
- **Read path**:
  - API composes from Redis (recent) and Elastic (historical) using keyset pagination.
  - Background workers compact old feed entries into cold storage (S3 + Presto) for analytics.
- **Deduplication**:
  - Use event IDs with Kafka's exactly-once semantics or de-dup store (Redis bloom filter) to avoid duplicates.
- **Backfill jobs**:
  - Replay from Kafka using consumer groups with offsets.
  - Snapshot DB to rebuild Elastic index. Use change data capture to maintain sync.
- **SLOs**:
  - Freshness under 2 seconds. Monitor pipeline lag. Add circuit breaker around Elastic queries to fall back to cached summary if cluster degraded.

### In-App Inbox and Alerts (Bullhorn-Style)
- **Use case**: notifications, chat-like updates, targeted alerts.
- **Components**:
  - Producer services publish events to Kafka topics (e.g., `alerts`, `messages`).
  - Notification service processes and fans out to WebSocket gateways, push notification service, email.
  - Store messages in scalable store (Cassandra, DynamoDB) partitioned by user ID + time bucket to support TTL and wide-row access.
- **Delivery semantics**:
  - Use sequence IDs per conversation to ensure ordering. Persist ack offsets per device to resume.
- **Fan-out patterns**:
  - Write-time fan-out for high priority (pre-compute per user). Read-time fan-out for large audiences (store broadcast messages once with membership lookup on fetch).
- **Real-time channels**:
  - WebSockets behind load balancer; use sticky sessions for connection state or external session store.
  - For offline delivery, utilize FCM/APNs. Include dedupe keys to avoid resending identical alerts.
- **Monitoring**:
  - Track delivery latency percentiles. Configure DLQs for failed template renders. Provide admin UI for replays.

### Job Scheduler at Scale (Clockwork-Like)
- **Problem**: schedule millions of jobs reliably (cron, one-off, recurring).
- **Architecture**:
  - Control plane handles job definitions, cron parsing. Store metadata in SQL or metadata service with strong consistency.
  - Sharded scheduler workers pull upcoming jobs. Each shard owns time ranges using consistent hashing to avoid duplication.
  - Use Redis or ZooKeeper for leader election and distributed locks (e.g., Redisson). Heartbeats detect worker failures and trigger shard rebalancing.
  - Execution plane: workers enqueue tasks onto message queues (Kafka, RabbitMQ, SQS) for execution microservices.
- **Reliability**:
  - Persist execution logs; implement at-least-once semantics, with idempotent job handlers to avoid double runs.
  - Support backpressure: if a downstream queue is full, scheduler delays job with exponential backoff.
- **Cron evaluation**:
  - Precompute next fire times using libraries (Quartz). For very large volumes, bucket jobs into minute-level wheels (hierarchical timing wheels) to reduce CPU.
- **APIs**:
  - Provide pause/resume, override, SLA monitoring. Alert when job misses window. Offer dry-run mode with simulated schedules.

### Metrics Platform for SLOs
- **Ingestion**:
  - Agents (sidecars, SDKs) push metrics to edge collectors (Prometheus remote-write, OTLP exporters). Collectors batch, compress, and forward to central ingest.
  - Use Kafka as durable buffer to decouple producers from storage. Partition by metric name for aggregation locality.
- **Storage**:
  - Time-series database with multi-tier: hot storage (Cortex/Mimir) for recent data; warm storage (Parquet on S3) for historical queries. Downsample older data (e.g., 5m rollups after 7 days).
- **Query Layer**:
  - Provide PromQL/Grafana. Implement query caching (memcached) for popular dashboards.
- **SLO calculations**:
  - Store error budget windows (4 hour, 30 day). Precompute rolling aggregates using Flink/Spark streaming jobs. Use burn-rate alerts to notify on fast consumption.
- **Multi-tenancy and cardinality**:
  - Enforce label cardinality quotas per team. Offer tooling to detect high-cardinality metrics automatically (topK by series count).
- **Observability**:
  - Instrument the observability stack itself (watchdog metrics). Provide anomaly detection for ingestion gaps.
- **Security**:
  - Tenant isolation with per-tenant API tokens. Encrypt in transit with mTLS. Apply RBAC for dashboard sharing.

---

## Next Steps
- Practice explaining these topics aloud; interviewers value clarity and structure.
- Implement the Java snippets in a local IDE and step through with a debugger to internalize flows.
- Build small Spring Boot services to experiment with resilience patterns and messaging setups.
- For system design, sketch component diagrams and rehearse trade-off discussions.
