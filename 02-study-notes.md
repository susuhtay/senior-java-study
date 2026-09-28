# Study notes, Java examples, and exercises

Use these units in the order assigned by the weekly plan. The snippets are teaching excerpts: add imports, supporting types, configuration, and tests in your practice project. Do not paste unrelated excerpts into one class. The [answer key](04-answer-key.md) is separate so you can test recall honestly.

## U1 · Core Java and the move beyond Java 8

**Understand.** Keep your existing knowledge of object identity, equality, exceptions, interfaces, streams, and resource management. Add records for data carriers, switch expressions for exhaustive state handling, and pattern matching where it makes branching clearer. `var` changes local type spelling, not static typing. Use `Instant` for event timestamps and inject `Clock` when business logic depends on time. Define money scale and rounding explicitly with `BigDecimal`.

A record makes component references final; it does not make referenced objects deeply immutable. A defensive copy protects this snapshot from later changes to the caller's list. [Official records tutorial](https://dev.java/learn/records/).

```java
record OrderSnapshot(UUID id, List<String> tags, Instant createdAt) {
    OrderSnapshot {
        Objects.requireNonNull(id);
        tags = List.copyOf(tags); // rejects null list/elements; String is immutable
        Objects.requireNonNull(createdAt);
    }
}

enum Status { RECEIVED, ACCEPTED, CANCELLED, RECONCILED }

static boolean canCancel(Status status) {
    return switch (status) {
        case RECEIVED, ACCEPTED -> true;
        case CANCELLED, RECONCILED -> false;
    };
}

BigDecimal total = new BigDecimal("125.40")
        .multiply(BigDecimal.valueOf(3));
```

**Production mistakes.** Using `double` for exact amounts; assuming `BigDecimal.equals` ignores scale; storing local wall-clock time without a zone policy; catching `Exception` and returning a success response; putting mutable lists into supposedly immutable DTOs. Streams with side effects often hide state changes and are harder to debug than a clear loop.

**Exercise, 50 minutes.** Convert one Java 8 style DTO into a record; prove that mutating the original list cannot change the record. Implement the cancellation decision with tests for every status. Add a time-based rule using a fixed `Clock`. Explain which changes cannot compile on Java 8.

**Self-test.** U1.1 Is a record containing `List<Order>` deeply immutable? U1.2 How do `new BigDecimal("1.0")` and `new BigDecimal("1.00")` compare with `equals` and `compareTo`? U1.3 Why inject `Clock` instead of calling `Instant.now()` throughout business code?

## U2 · Collections, generics, and complexity

**Understand.** Choose a collection by access patterns: an `ArrayList` for indexed iteration, a `HashMap` for expected constant-time lookup, a `TreeMap` for ordered keys, and an `ArrayDeque` for a queue/stack. An in-memory map is not durable duplicate protection. Keys must have stable equality/hash behavior while stored. Generic types are invariant: `List<Trade>` is not a subtype of `List<Object>`. For flexible APIs, a source that produces `T` can use `? extends T`; a destination consuming `T` can use `? super T`. [Collections](https://dev.java/learn/api/collections-framework/), [generics](https://dev.java/learn/generics/).

```java
record Trade(String symbol, long quantity) {}

static Map<String, Long> quantities(List<Trade> trades) {
    Map<String, Long> result = new HashMap<>();
    for (Trade trade : trades) {
        result.merge(trade.symbol(), trade.quantity(),
                (left, right) -> Math.addExact(left, right));
    }
    return Map.copyOf(result);
}

static <T> void append(List<? extends T> source, List<? super T> target) {
    target.addAll(source);
}
```

The aggregation is O(n) expected time and O(k) space for k symbols. Overflow fails explicitly rather than silently wrapping. Neither `HashMap` nor this method's surrounding business workflow becomes thread-safe merely because the return value is unmodifiable.

**Production mistakes.** Mutating keys; relying on `HashMap` iteration order; using raw types and casts; loading millions of database rows to group in Java; assuming `Collections.unmodifiableList` copies its backing list; using `parallelStream` around blocking HTTP calls without capacity control.

**Exercise, 50 minutes.** Aggregate 100,000 synthetic trades, including duplicate symbols and an overflow case. Compare a loop with a stream for clarity. Demonstrate a failed lookup after mutating a key. Write one generic helper without unchecked casts. Move aggregation to SQL when the data already lives in PostgreSQL and compare data transferred.

**Self-test.** U2.1 Why can a mutable map key become unfindable? U2.2 Why can you read a `Number` from `List<? extends Number>` but not add an arbitrary `Integer`? U2.3 When should the aggregation run in SQL rather than Java?

## U3 · Spring Boot and application boundaries

**Understand.** Spring manages objects and wires their dependencies. Boot adds conditional configuration and conventions based on your dependencies and properties; it does not remove the need to understand what is configured. Keep HTTP concerns in controllers, use cases and transaction boundaries in services, and database access in repositories. Prefer constructor injection so dependencies are explicit and ordinary unit tests can construct the class. [Build a REST service with Spring](https://spring.io/guides/gs/rest-service/).

```java
@Service
class OrderQueryService {
    private final OrderRepository orders;

    OrderQueryService(OrderRepository orders) {
        this.orders = orders;
    }

    @Transactional(readOnly = true)
    OrderView get(UUID id, String accountId) {
        var order = orders.findByIdAndAccountId(id, accountId)
                .orElseThrow(() -> new OrderNotFound(id));
        return OrderView.from(order); // map while persistence context is open
    }
}
```

This method is called through the Spring bean from another component. The project supplies the repository and DTO. Querying by both ID and account helps enforce ownership rather than merely checking that the caller has logged in.

**Production mistakes.** Business rules inside controllers; field injection that hides dependencies; component scanning that misses a package; manual `new` of a class that needs Spring interception; committing credentials in properties; using development profiles in production.

**Exercise, 60 minutes.** Trace a request through controller → service → repository. Swap a repository implementation in a plain unit test. Externalize one non-secret setting. Break a required property deliberately and make startup fail with an understandable message. Explain one auto-configuration decision from startup diagnostics.

**Self-test.** U3.1 What does Boot add to Spring? U3.2 Why is constructor injection useful for tests? U3.3 Why can manually creating a service bypass `@Transactional` behavior?

## U4 · Designing and building REST APIs

**Understand.** Design the request, response, status, and retry semantics before annotations. Separate transport DTOs from JPA entities. Validate syntax at the boundary and business rules inside the service. Return 201 plus a resource location for a newly created order, 400 for invalid input, 404 for an unavailable resource, and 409 for a state conflict. A consistent error body helps callers respond without parsing exception messages; Spring supports `ProblemDetail`. [Spring error responses](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-rest-exceptions.html).

```java
record CreateOrderRequest(
        @NotBlank @Size(max = 16) String symbol,
        @Positive long quantity,
        @NotNull @DecimalMin("0.0001") BigDecimal limitPrice) {}

// Inside a @RestController; account identity comes from authenticated context.
@PostMapping("/orders")
ResponseEntity<OrderView> create(@Valid @RequestBody CreateOrderRequest input) {
    OrderView result = service.create(currentAccount.id(), input);
    return ResponseEntity.created(URI.create("/orders/" + result.id()))
            .body(result);
}

// Inside a @RestControllerAdvice; logging belongs in a separate concern.
@ExceptionHandler(OrderConflict.class)
ProblemDetail conflict(OrderConflict error) {
    return ProblemDetail.forStatusAndDetail(
            HttpStatus.CONFLICT, "Order state does not allow this operation");
}
```

This first-version POST has no retry protection; Week 7 adds the idempotency contract. List endpoints need a bounded page size and deterministic ordering, for example `(createdAt, id)`. API compatibility includes behavior and defaults, not just field names.

**Production mistakes.** Returning HTTP 200 for every outcome; accepting arbitrary page sizes; exposing entity relationships or stack traces; trusting a request's `accountId`; assuming POST is safe to retry; changing enum values without considering existing consumers.

**Exercise, 90 minutes across sessions.** Implement create/get/list/cancel, a maximum page size of 100, validation, and consistent error responses. Save five HTTP examples. Add tests proving that malformed input cannot reach persistence. Write the repeat-cancel behavior explicitly.

**Self-test.** U4.1 What is the difference between 400 and 409 in this project? U4.2 Why avoid returning a JPA entity directly? U4.3 Why does pagination need deterministic ordering?

## U5 · Calling APIs and handling remote failure

**Understand.** An HTTP client needs bounded waiting, error translation, and a contract for empty or invalid responses. Use the synchronous `RestClient` for this MVC project. `WebClient` is useful when you deliberately choose reactive/nonblocking composition. In Spring Framework 7, `RestTemplate` is deprecated; learn to recognize it in legacy applications while writing new examples with `RestClient`. [Official REST clients reference](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html).

```java
// Build once during configuration; keep this client for the application's life.
var http = HttpClient.newBuilder()
        .connectTimeout(Duration.ofSeconds(1))
        .build();
var factory = new JdkClientHttpRequestFactory(http);
factory.setReadTimeout(Duration.ofSeconds(2));
RestClient client = RestClient.builder()
        .baseUrl("http://localhost:9090") // controlled mock service in this lab
        .requestFactory(factory)
        .build();

InstrumentView instrument = client.get()
        .uri("/instruments/{symbol}", symbol)
        .retrieve()
        .onStatus(status -> status.value() == 404,
                (request, response) -> { throw new UnknownInstrument(symbol); })
        .body(InstrumentView.class);
if (instrument == null) {
    throw new InvalidUpstreamResponse("Missing instrument body");
}
```

The application must map transport failures, other error statuses, and JSON-decoding failures deliberately. Connection and read timeouts are not a complete business deadline across multiple attempts. A timeout means the caller does not know the result; it does not prove the server did no work. See the [JDK request-factory API](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/http/client/JdkClientHttpRequestFactory.html) for its timeout configuration.

**Production mistakes.** Calling an external service while holding a database lock; retrying all exceptions; creating a new connection pool per request; accepting a caller-controlled URL; turning a failed lookup into a valid zero/default value; logging tokens or entire sensitive payloads.

**Exercise, 90 minutes.** Use a local stub to return success, 404, 500, malformed JSON, an empty body, and a delayed response. Translate each to a domain outcome. Record elapsed time for the timeout case. Do not add automatic POST retries until you define and test idempotency.

**Self-test.** U5.1 Does a timeout prove an order was not created remotely? U5.2 Why are remote calls inside a database transaction risky? U5.3 Which failures would you consider retrying, and what limits must apply?

## U6 · SQL and query performance

**Understand.** Use schema constraints to protect invariants regardless of which code path writes data. Choose indexes from query predicates and ordering, then measure. A plan's estimate is not measured execution; `EXPLAIN ANALYZE` executes the statement. Use a controlled lab for writes. Review returned rows, rows scanned, buffers, sorting, and table statistics before assuming an index will help. [PostgreSQL EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html).

```sql
CREATE TABLE orders (
  id uuid PRIMARY KEY,
  account_id varchar(80) NOT NULL,
  symbol varchar(16) NOT NULL,
  quantity bigint NOT NULL CHECK (quantity > 0),
  limit_price numeric(19,4) NOT NULL CHECK (limit_price > 0),
  status varchar(20) NOT NULL,
  created_at timestamptz NOT NULL,
  version bigint NOT NULL DEFAULT 0
);
CREATE INDEX orders_account_created_idx
  ON orders (account_id, created_at DESC, id DESC);
```

```java
// NamedParameterJdbcTemplate: values are bound, not concatenated into SQL.
var rows = jdbc.query("""
        SELECT id, symbol, quantity
        FROM orders WHERE account_id = :account
        ORDER BY created_at DESC, id DESC LIMIT :limit
        """, Map.of("account", accountId, "limit", 50), orderSummaryMapper);
```

**Production mistakes.** Indexing every column; using unbounded queries; concatenating sort field names from user input instead of allowlisting them; judging performance on 20 rows; ignoring write costs; migrating a schema without checking old application compatibility.

**Exercise, 75 minutes.** Seed 100,000 synthetic rows with uneven account distribution. Compare the list query before and after the proposed index using the same parameters. Save plans and elapsed times. Explain one case where the planner reasonably prefers a sequential scan. Apply schema changes with versioned migrations, not ad hoc manual edits.

**Self-test.** U6.1 Why might PostgreSQL ignore an index? U6.2 Which invariant belongs in both Java validation and a database constraint? U6.3 Why does a faster read not automatically justify an additional index?

## U7 · Transactions, Hibernate, and persistence

**Understand.** A database transaction groups changes into one commit or rollback. Isolation controls concurrent visibility; it does not automatically enforce every business invariant. PostgreSQL's default Read Committed isolation can return different committed data in two statements of one transaction. Use version checks, conditional updates, row locks, or stronger isolation according to the invariant. [PostgreSQL isolation](https://www.postgresql.org/docs/current/transaction-iso.html).

Hibernate tracks managed entities and flushes changes; fetching relationships carelessly can create N+1 queries. Prefer an explicit fetch plan or DTO query for each use case. [Spring Data JPA query methods](https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html).

```java
// Entity fragment: use jakarta.persistence annotations with Boot 4.
@Version
private long version;

// Separate Spring service method, called through its proxy.
@Transactional
public void cancel(UUID id, String accountId) {
    var order = orders.findByIdAndAccountId(id, accountId)
            .orElseThrow(() -> new OrderNotFound(id));
    order.cancel(); // checks permitted transition; modifies managed entity
    audit.save(AuditEntry.cancelled(id, accountId));
}
```

With one database/transaction manager, the status and audit insert commit together. Concurrent stale entity updates fail through optimistic locking; convert that into a documented conflict response. An annotation on a self-invoked method does not start the expected proxy transaction. By default, unchecked exceptions cause rollback; checked exceptions need an appropriate rollback rule if they must do so. [Spring transaction annotations](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html).

**Production mistakes.** Catching an exception and then committing partial work; assuming a DB transaction includes Kafka or HTTP; `EAGER` everywhere; Open Session in View hiding lazy fetches; paginating a collection fetch join without examining SQL; using `readOnly` as an authorization control.

**Exercise, 100 minutes across sessions.** Set `spring.jpa.open-in-view=false`. Force audit insertion to fail and verify the order update rolls back. Open two independent transactions, load the same order, and prove one stale update is rejected. Inspect SQL for a list endpoint, then eliminate N+1 behavior without fetching the entire database.

**Self-test.** U7.1 Why can `this.cancel()` bypass transactional interception? U7.2 What does `@Version` prevent, and what does it not prevent? U7.3 Why can a transaction that updates an order and calls HTTP still leave inconsistent systems?

## U8 · Security and ownership

**Understand.** Authentication identifies a caller; authorization decides what that caller may do to a particular resource. A JWT needs signature, issuer, time, and intended-audience validation. A valid token alone does not grant access to another account's order. Use the identity provider's contract to map claims to authorities. [Spring JWT resource server](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html).

```java
// Security configuration excerpt for an API using Authorization bearer tokens
// only: no session/cookie-based authentication or browser-managed credentials.
@Bean
SecurityFilterChain apiSecurity(HttpSecurity http) throws Exception {
    return http
        .csrf(csrf -> csrf.disable()) // valid only under the assumption above
        .sessionManagement(s -> s.sessionCreationPolicy(
                SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(a -> a
            .requestMatchers(HttpMethod.GET, "/orders", "/orders/**")
                .hasAuthority("SCOPE_orders.read")
            .requestMatchers(HttpMethod.POST, "/orders", "/orders/**")
                .hasAuthority("SCOPE_orders.write")
            .anyRequest().denyAll())
        .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))
        .build();
}
```

Configure the trusted issuer and audience validator as part of the resource server setup; the filter-chain excerpt alone is not complete token configuration. Keep the service ownership predicate even when HTTP scopes pass. Method authorization can protect non-HTTP entry points; it requires enabling method security and using the Spring proxy. [Method security](https://docs.spring.io/spring-security/reference/servlet/authorization/method-security.html).

**Production mistakes.** Trusting a decoded but unverified token; checking role but not resource ownership; treating CORS as authorization; disabling CSRF for cookie-authenticated applications; exposing management endpoints publicly; returning secrets in logs; assuming UUIDs make authorization unnecessary.

**Exercise, 100 minutes.** Test no token, expired token, wrong issuer/audience, insufficient scope, valid owner, and a different account. Mock-JWT tests are useful for authorization but do not prove signature validation: add a real signed-token verification test with a controlled test key/issuer. Document whether hidden resources return 404 or 403, then apply the policy consistently.

**Self-test.** U8.1 How do authentication, scope checks, and ownership checks differ? U8.2 Why do mock-JWT controller tests not prove cryptographic validation? U8.3 When would the CSRF assumption in this example be invalid?

## U9 · Automated testing and useful confidence

**Understand.** A unit test isolates a business rule. An HTTP/controller test checks routing, validation, status, serialization, and relevant filters. A repository integration test checks actual SQL/schema behavior. A small full-service test checks the assembled application. Mocking the repository cannot prove transaction isolation or database constraints. Use PostgreSQL through Testcontainers for those checks. [Boot testing](https://docs.spring.io/spring-boot/reference/testing/index.html), [PostgreSQL Testcontainers module](https://java.testcontainers.org/modules/databases/postgres/).

```java
// JUnit Jupiter test excerpt; Order is your domain object, no Spring needed.
@Test
void reconciledOrderCannotBeCancelled() {
    var order = Order.fixture(Status.RECONCILED);

    assertThrows(OrderConflict.class, order::cancel);
    assertEquals(Status.RECONCILED, order.status());
}

@ParameterizedTest
@EnumSource(value = Status.class, names = {"RECEIVED", "ACCEPTED"})
void activeOrderCanBeCancelled(Status initial) {
    var order = Order.fixture(initial);
    order.cancel();
    assertEquals(Status.CANCELLED, order.status());
}
```

Write deterministic tests with fixed clocks, explicit fixtures, and isolated state. For asynchronous outcomes, use bounded polling or coordination primitives instead of guessed sleeps. The current JUnit documentation covers the Jupiter APIs; use the version managed by the Boot build. [JUnit guide](https://docs.junit.org/current/user-guide/).

**Production mistakes.** Testing getters instead of business behavior; mocking everything; running all tests with a full Spring context; relying only on H2 for PostgreSQL-specific behavior; wrapping a test in a transaction that hides real commit-time failures; accepting flaky tests as normal.

**Exercise, 120 minutes across sessions.** Implement one unit test, one API validation test, one PostgreSQL rollback test, and one duplicate-request test. Temporarily introduce a defect in cancellation logic and verify a meaningful test fails. Ensure the normal CI command actually runs integration tests; configure the relevant Maven lifecycle instead of assuming a file named `*IT` runs by default.

**Self-test.** U9.1 Which test should prove a unique constraint works? U9.2 Why is 90% line coverage insufficient evidence? U9.3 How can a test-managed transaction hide a production bug?

## U10 · Debugging, JVM internals, and performance

**Understand.** Class loading brings bytecode into the runtime; the interpreter and JIT execute and optimize it. Objects generally occupy heap memory; thread execution uses stacks, and the JVM also needs native memory, metadata, and compiled-code storage. Garbage collection reclaims unreachable objects, not objects your application still retains accidentally. A large heap cannot fix every memory problem.

Debug with a hypothesis: identify the affected request, time window, change, and dependency; capture evidence; reproduce; change one cause; verify the same workload. CPU saturation, lock waiting, connection-pool waiting, and downstream latency require different remedies. JFR helps connect allocation, CPU, and waiting behavior. [JFR introduction](https://dev.java/learn/jvm/jfr/getting-started/).

```java
// Deliberately defective teaching example: every request is retained forever.
final class RequestHistory {
    private final List<byte[]> payloads = new ArrayList<>();
    synchronized void remember(byte[] requestBody) {
        payloads.add(requestBody.clone());
    }
}
```

Reproduce with a small lab load. Explain the retention chain before changing the collector. A sensible fix may be to remove payload retention entirely, keep bounded metadata, or persist audit data under an explicit retention policy.

```bash
# Replace 12345 with the PID of your own study application.
jcmd 12345 Thread.print
jcmd 12345 JFR.start name=study settings=profile duration=60s filename=study.jfr
```

Check command availability with `jcmd <pid> help`. Diagnostic commands have different overhead; heap dumps/histograms are not interchangeable with a short recording. [Official jcmd reference](https://docs.oracle.com/en/java/javase/25/docs/specs/man/jcmd.html).

**Production mistakes.** Treating every latency spike as GC; setting heap equal to a container's entire memory limit; benchmarking before warm-up; comparing different data/workloads; quoting only averages; fixing a symptom without a regression check.

**Exercise, 100 minutes across sessions.** Seed the project, warm it up, run the same workload three times, and record throughput, errors, p50/p95/p99, CPU, heap, and DB pool usage. Diagnose either N+1 reads or deliberate object retention. Save before/after evidence and the remaining bottleneck. Avoid tuning GC flags without measurements supporting a GC problem.

**Self-test.** U10.1 Why can low CPU coexist with high request latency? U10.2 Why can GC not reclaim the arrays in this example? U10.3 What must remain comparable in a before/after performance claim?

## U11 · Concurrency and virtual threads

**Understand.** Atomicity, visibility, and ordering are different concerns. `volatile` can make a value visible but does not make `count++` atomic. Locks can protect a compound invariant if every participant uses the same lock. A concurrent collection makes documented operations safe; it does not make a multi-step workflow atomic. Prefer atomic map operations for local single-key changes. [Java concurrency APIs](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/package-summary.html), [ConcurrentHashMap](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html).

```java
final ConcurrentHashMap<String, LongAdder> calls = new ConcurrentHashMap<>();

void recordCall(String boundedRouteName) {
    calls.computeIfAbsent(boundedRouteName, key -> new LongAdder()).increment();
}

// One shared limiter per application instance, not one per request.
final Semaphore vendorSlots = new Semaphore(20);

InstrumentView boundedLookup(String symbol) throws InterruptedException {
    if (!vendorSlots.tryAcquire(100, TimeUnit.MILLISECONDS)) {
        throw new DependencyBusy();
    }
    try {
        return instrumentClient.lookup(symbol); // separately configured timeout
    } finally {
        vendorSlots.release();
    }
}
```

Virtual threads are useful for many concurrent blocking I/O operations, not for making CPU-bound computation faster. They do not expand your database's connection limit or a vendor's quota. Use a virtual-thread-per-task executor for the lab, close it appropriately, preserve interruption, and still bound admitted work. Do not pool virtual threads to imitate a platform-thread pool. [JEP 444](https://openjdk.org/jeps/444).

Version distinction: Java 21 can pin carrier threads during some synchronized blocking operations; JEP 491, delivered in JDK 24, removes the monitor-related pinning limitation. Do not repeat the Java 21 blanket advice as if it describes JDK 25. Other blocking/native cases still deserve measurement. [JEP 491](https://openjdk.org/jeps/491).

**Production mistakes.** Swallowing interruption; using the common fork-join pool for blocking vendor calls; unbounded queues; holding locks during network I/O; assuming one JVM's lock protects another replica; using `LongAdder` as an exact transactional balance.

**Exercise, 100 minutes.** Reproduce a lost increment, then fix it. Launch 100 simulated lookups with a virtual-thread executor and verify no more than 20 enter the dependency concurrently. Test rejection/wait limits. Run two app instances and explain why duplicate-order protection must live in shared storage.

**Self-test.** U11.1 Why does `volatile int count` not fix `count++`? U11.2 Why might virtual threads leave throughput unchanged? U11.3 Why does the semaphore limit change when you add another replica?

## U12 · Clean code, design patterns, and refactoring

**Understand.** Clean code exposes domain intent, dependencies, and failure behavior. Refactoring changes structure while preserving agreed behavior; tests protect that boundary. Patterns are tools for a recurring problem, not goals. Use Strategy for alternative policies, Adapter to isolate a vendor API, and Repository to separate persistence access. Avoid a generic abstraction until two concrete cases show what actually varies.

```java
interface CancellationPolicy {
    boolean allows(OrderSnapshotWithStatus order);
}

final class StandardCancellationPolicy implements CancellationPolicy {
    @Override
    public boolean allows(OrderSnapshotWithStatus order) {
        return order.status() == Status.RECEIVED
                || order.status() == Status.ACCEPTED;
    }
}

// Service excerpt. The state transition remains a domain operation.
if (!policy.allows(order.snapshot())) {
    throw new OrderConflict("Cancellation is not allowed");
}
order.cancel();
```

Use the interface only if policies truly vary, such as a market-close rule versus an internal simulation rule. One small method is simpler when there is only one policy. In review, prioritize correctness, comprehensibility, and maintainability over personal style preferences. [Google engineering code-review guidance](https://google.github.io/eng-practices/review/reviewer/).

**Production mistakes.** A service that knows HTTP, SQL, and every business rule; catching and translating exceptions repeatedly; naming technical steps rather than business actions; changing behavior during a large untested rewrite; adding a factory/strategy layer for a single constant choice.

**Exercise, 90 minutes.** Take a long cancellation method; first capture its behavior in tests, then extract validation and persistence boundaries in small steps. Introduce a vendor adapter with a fake implementation for tests. Write a review comment explaining the practical benefit of one change and explicitly label one style suggestion as optional.

**Self-test.** U12.1 When is Strategy useful, and when is it unnecessary? U12.2 How do you distinguish refactoring from a behavior change? U12.3 Why isolate a remote vendor behind an adapter?

## U13 · System design and distributed systems

**Understand.** Begin with users, operations, load, data retention, invariants, and acceptable failure. Choose the simplest architecture that satisfies those constraints. A modular monolith keeps deployment and transactions manageable while preserving domain boundaries. Split services when independent ownership, scaling, or reliability needs justify the cost.

Across a network, failures can be partial: a request can succeed remotely while the response is lost. Decide which operations need strong consistency and where stale or delayed data is acceptable. During a network partition, guarantees about consistent reads/writes can conflict with always serving requests; stating “choose two of CAP” without a concrete operation is not a design.

```java
record CreateOrderCommand(String accountId, String idempotencyKey,
                          String symbol, long quantity, BigDecimal price) {}

// Illustrative service contract; implementation must be backed by DB constraints.
interface OrderSubmission {
    SubmissionResult submit(CreateOrderCommand command);
}
```

Define the key's scope as `(accountId, idempotencyKey)`, the normalized request fingerprint, the retained result, and the retention window. In one database transaction, claim the key and create the order/result. Same key + same request returns the original result; same key + different request conflicts. A check-then-insert without a database constraint races. Use PostgreSQL `INSERT ... ON CONFLICT` or another intentional conflict protocol, not catching a constraint exception and continuing in an already-failed transaction.

**Production mistakes.** Drawing boxes before stating requirements; starting with ten microservices; promising exactly-once external effects without a protocol; omitting retention and duplicate windows; assuming a load balancer solves database contention.

**Exercise, 90 minutes.** Design for a synthetic 50 requests/second baseline and a fivefold peak. At 200 ms average response time, Little's Law suggests roughly 10 requests in flight at baseline; this is a first estimate, not a pool-size prescription. Draw synchronous and asynchronous paths, identify the source of truth, and write one consistency decision for order creation versus instrument metadata. Review the [transactional outbox pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html) before U14.

**Self-test.** U13.1 When would you split this monolith? U13.2 How do you distinguish a duplicate request from accidental reuse of a key? U13.3 What can happen after a client times out but the database commits?

## U14 · Messaging, outbox, and reconciliation

**Understand.** A broker decouples producer and consumer timing. It also introduces duplicate delivery, delayed processing, partitioning, schema evolution, and operational backlog. Kafka orders records within a partition; choose an aggregate key when you need related records routed together. Kafka's transactional guarantees do not automatically include arbitrary PostgreSQL side effects. [Kafka design](https://kafka.apache.org/43/design/design/).

An outbox stores an event in the same database transaction as the business change. A relay publishes it later. The relay can crash after publish but before marking completion, so consumers must tolerate duplicates. [AWS outbox guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html).

```java
// Executed through a Spring proxy. Both repositories use the same database.
@Transactional
public void accept(UUID orderId) {
    var order = orders.getRequired(orderId);
    order.accept();
    long sequence = order.advanceEventSequence(); // durable aggregate sequence
    outbox.save(OrderEvent.accepted(order.id(), sequence));
}

// Consumer application service; the listener commits/acks only AFTER return.
@Transactional
public void apply(OrderEvent event) {
    // INSERT ... ON CONFLICT DO NOTHING, unique (consumer_name, event_id).
    if (processedEvents.insertIfAbsent("reconciliation", event.id()) == 0) {
        return;
    }
    reconciliation.apply(event); // DB effect commits with deduplication marker
}
```

The aggregate's event sequence is a separate persisted field, advanced in the same transaction; it is not a guess at Hibernate's next optimistic-lock version. For out-of-order distinct events, deduplication alone is insufficient: enforce sequence/state rules or re-read the authoritative order. Configure listener acknowledgments intentionally and test the chosen mode. [Spring Kafka listener containers](https://docs.spring.io/spring-kafka/reference/kafka/receiving-messages/message-listener-container.html).

**Production mistakes.** Publishing then committing the DB as separate uncoordinated steps; acknowledging before the DB commits; marking an event processed before an effect in another transaction; infinite poison-message retries; no backlog-age metric; treating a dead-letter queue as resolution.

**Exercise, 120 minutes across sessions.** Start with one relay to simplify ordering. Kill it after publish but before marking sent. Redeliver an event 20 times and prove one durable reconciliation effect. Introduce a malformed event, quarantine it with a reason, fix the cause, and replay deliberately. Compare source orders with reconciliation records to detect missing results.

**Self-test.** U14.1 Which failure window does an outbox close? U14.2 Why can duplicate events still occur? U14.3 Where must the deduplication marker and business effect commit?

## U15 · Caching and reliability controls

**Understand.** Cache-aside reads the cache, loads the source on a miss, then stores a result. Decide what staleness is acceptable and how entries expire or invalidate. Cache instrument display metadata for this project; use the database for authoritative order state and authorization. Spring's cache annotation abstracts access, but expiry is a cache-provider configuration. [Spring caching](https://docs.spring.io/spring-framework/reference/integration/cache/annotations.html), [Redis expiry](https://redis.io/docs/latest/commands/expire/).

```java
// On a separate proxied bean; normalize/validate symbol before this call.
@Cacheable(cacheNames = "instrumentDisplay", key = "#symbol", sync = true)
public InstrumentDisplay getDisplay(String symbol) {
    return referenceDataRepository.getDisplay(symbol);
}
```

Configure a 60-second TTL for the lab with the selected provider. `sync=true` is provider-dependent coordination and is not a general distributed lock. A cache miss stampede, Redis outage, or multi-replica inconsistency still needs a policy. Database fallback is acceptable only within its capacity.

Timeouts bound waiting, retries repeat selected operations, backoff/jitter spread retries, circuit breakers stop calls during repeated failure, and bulkheads limit concurrency. They solve different problems. A retry budget must fit the caller's deadline and must not multiply at every layer. [AWS guidance on timeouts and retries](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/).

**Production mistakes.** Caching permissions or balances without a consistency design; retrying a timed-out write with a new key; serving stale data without saying so; caching errors as successful results; choosing very long TTLs without invalidation; adding resilience annotations without testing actual failure behavior.

**Exercise, 100 minutes.** Prove a cached read reduces source lookups, then update source data and measure stale duration. Stop Redis and demonstrate the documented fallback or bounded failure. Delay a dependency, cap attempts, and show maximum in-flight calls and response time. Explain why a fallback cannot fabricate a successful order acceptance.

**Self-test.** U15.1 Does `@Cacheable` alone set a TTL? U15.2 How do a circuit breaker and a timeout differ? U15.3 When can retries worsen an outage?

## U16 · Docker and CI/CD

**Understand.** A container image packages the runtime and application; a container is a running instance with process, network, and filesystem boundaries. Persist database data outside the application container. Build reproducibly, run with a non-root identity, expose only required ports, and separate environment configuration from the image. [Docker Java guide](https://docs.docker.com/guides/java/).

```dockerfile
# Illustrative runtime image; pin an actual image digest in your project.
FROM eclipse-temurin:25-jre
WORKDIR /app
COPY --chown=10001:10001 target/order-service.jar /app/app.jar
USER 10001:10001
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

Configure Maven to produce the named executable JAR. This image expects writable temporary storage supplied by the runtime; keep application state in PostgreSQL. Budget memory for the whole JVM, not only heap.

**Java lifecycle connection:** use Boot's shutdown lifecycle to let in-flight work finish within a deadline, and ensure custom executors stop accepting new work. Test `SIGTERM`; a successful `java -jar` start says nothing about safe termination. [Boot graceful shutdown](https://docs.spring.io/spring-boot/reference/web/graceful-shutdown.html).

Your CI stages are checkout → configured JDK → `./mvnw -B verify` → image build → publish tagged artifact → deploy sandbox → smoke checks. Unit and integration tests must be bound to that command. Use immutable image identifiers and the same artifact across environments. For GitHub Actions, start from the current official Maven workflow and pin actions according to your repository's policy. [GitHub Java/Maven CI](https://docs.github.com/en/actions/tutorials/build-and-test-code/java-with-maven).

**Production mistakes.** Baking secrets into layers; using mutable `latest` as the only release identity; skipping integration tests in CI; making database changes incompatible with the previous app; rebuilding a different binary for production; relying on local IDE settings.

**Exercise, 120 minutes across sessions.** Build and run from a fresh checkout. Run tests in CI with PostgreSQL. Stop the app during a request, restart it, and verify persisted state. Release v2 with an additive DB migration, then roll the application back to v1. Explain why rollback becomes unsafe after a destructive schema migration.

**Self-test.** U16.1 What is the difference between an image and a container? U16.2 Why must database compatibility be considered before rollback? U16.3 What makes “works on my laptop” insufficient deployment evidence?

## U17 · Cloud deployment and observability

**Understand.** Cloud deployment adds identities, network rules, secrets, resource limits, health checks, storage, rollout policy, and cost. For one practice route, use AWS ECS/Fargate with a managed PostgreSQL database, a registry, and a controlled HTTPS entry point. Use the smallest suitable sandbox and tear it down after the exercise; no free-tier availability is assumed. [Official Fargate walkthrough](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/getting-started-fargate.html).

Logs describe events, metrics quantify behavior over time, and traces connect work across boundaries. Instrument the user-visible operations and failure paths. Avoid high-cardinality labels such as order IDs; those belong in appropriately redacted logs/traces. Boot uses Micrometer observations; choose one trace export path to avoid duplicate instrumentation. [Boot observability](https://docs.spring.io/spring-boot/reference/actuator/observability.html), [OpenTelemetry Java](https://opentelemetry.io/docs/languages/java/).

```java
// ObservationRegistry is injected; supporting service types are project code.
OrderView observedCreate(CreateOrderRequest input) {
    return Observation.createNotStarted("orders.create", observationRegistry)
            .lowCardinalityKeyValue("operation", "create")
            .observe(() -> orderService.create(input));
}
```

This intentionally omits order IDs as metric tags. Add a correlation mechanism, error classification, DB-pool metrics, request percentiles, and outbox backlog age. Secure management endpoints separately; the U8 deny-all rule must be extended deliberately for the runtime's limited health access. Liveness should identify a process needing restart; tying it to every shared dependency can cause restart storms. [Actuator endpoints](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html).

An SLI is a measured indicator; an SLO is its target over a defined window. For the lab, define success as an eligible request producing the expected service outcome, and report intentional client errors separately. A short load test cannot prove a month's availability. [Google SRE service-level objectives](https://sre.google/sre-book/service-level-objectives/).

**Production mistakes.** No timeout or resource ceiling; database reachable from the public internet without need; long-lived credentials in source; health returning green while all useful work fails; alerts without an owner/action; treating average latency as tail latency; not deleting sandbox resources.

**Exercise, 120 minutes across sessions.** Deploy one image, inject settings/secrets, execute smoke tests, follow one request across service and dependency, and trigger one actionable alert. Roll back the image and confirm database compatibility. If cloud access/budget is unavailable, do the local rehearsal and mark the actual cloud deployment gate outstanding.

**Self-test.** U17.1 How do logs, metrics, and traces help with a slow request? U17.2 Why avoid an order-ID metric label? U17.3 Why should a downstream outage not automatically fail liveness?

## U18 · Senior responsibilities: decisions, reviews, mentoring, tradeoffs

**Understand.** Senior work includes making a decision under uncertainty, identifying failure modes early, helping others deliver, and communicating the impact of choices. An ADR records context, alternatives, decision, consequences, and what evidence would make you revisit it. A review comment should identify a concrete risk and a proportionate improvement. Mentoring checks the other person's understanding rather than doing all the work for them. [Google code-review guidance](https://google.github.io/eng-practices/review/reviewer/).

```java
// Deliberately flawed review exercise.
public OrderView create(CreateOrderRequest request, String clientKey) {
    if (!orders.existsByClientKey(clientKey)) {
        return OrderView.from(orders.save(Order.from(request)));
    }
    return OrderView.from(orders.findByClientKey(clientKey));
}
```

**Example review:** “Two concurrent requests can both pass this check and create separate orders. Please make the account-scoped key unique in the database and add a concurrent test. We also need to compare the request fingerprint so the same key cannot return an unrelated result.” This identifies risk, mechanism, and evidence without criticizing the author.

**Example tradeoff:** “I chose a modular monolith because one team owns the workflow and the order/audit update benefits from one transaction. We accept coupled deployment for now. I would revisit a separate reconciliation service if its workload or release cadence becomes independently constrained.”

**Production mistakes.** Giving opinions without constraints; approving happy-path-only code; insisting every suggestion is blocking; inventing impact metrics; taking over a mentee's keyboard; reporting only technical detail during an incident rather than user impact and next action.

**Exercise, 90 minutes split across Sundays.** Write one ADR comparing optimistic and pessimistic locking. Review the code above, then propose the smallest corrective change and its test. Teach idempotency in five minutes using a retry timeline, ask the learner to explain it back, and record what confused them. Draft a 100-word incident update with impact, known facts, mitigation, and next update time.

**Self-test.** U18.1 What makes an ADR useful six months later? U18.2 Which parts of the example review are blocking, and which might be optional? U18.3 How do you demonstrate mentoring and leadership without claiming authority you did not have?
