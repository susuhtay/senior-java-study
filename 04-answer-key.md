# Separate answer key

Attempt the questions in [the study notes](02-study-notes.md) before reading this file. The official sources are linked beside each unit's explanation. Answers below describe the key points; your wording can differ.

Score each answer 0 = wrong/missing, 1 = partly correct, 2 = correct with an example or failure mode. For each unit, 5–6/6 is the recall target. Practical competence still requires its project exercise.

## U1 · Core Java

**U1.1** No. A record's component references are final, but a referenced list or its elements can still be mutable. A defensive copy such as `List.copyOf` prevents structural mutation through the original list, but mutable `Order` elements still need their own immutable representation or copy strategy.

**U1.2** `equals` returns false because scale differs; `compareTo` returns zero because the numeric values are equal. Decide whether scale is meaningful in your domain. Normalize inputs deliberately when producing an idempotency fingerprint so harmless formatting differences follow the documented policy.

**U1.3** An injected clock makes the time dependency explicit and permits deterministic boundary tests. With a fixed clock, you can test just before and after a deadline without waiting or depending on the machine's current time. Use an appropriate time zone when converting instants into business dates.

## U2 · Collections and generics

**U2.1** A hash-based map chooses a bucket using the key's hash. If fields involved in hashing/equality change after insertion, lookup may search a different bucket or fail equality checks. Use immutable keys such as a UUID or a record of immutable key components.

**U2.2** The list may really be a `List<Double>`. Reading as `Number` is safe, but adding an `Integer` would break that actual type. With `? super Integer`, adding an `Integer` is safe, although values read from it are only guaranteed to be `Object`.

**U2.3** Prefer SQL when the rows already live in the database, the aggregation maps naturally to SQL, and it reduces transferred data. Java may be appropriate for already-loaded bounded data or domain calculations unsuitable for SQL. Compare plans, data volume, and maintainability instead of choosing by habit.

## U3 · Spring Boot

**U3.1** Boot supplies conventions, dependency coordination, conditional auto-configuration, executable application packaging, and operational integration. Spring still provides dependency injection and many programming abstractions. Boot's configuration is conditional, not magic; inspect what beans and properties are actually active.

**U3.2** Required dependencies are visible and can be final. A unit test can pass a fake repository directly into the constructor without starting Spring or using reflection. It also makes an overly large dependency list visible, which can reveal an overburdened class.

**U3.3** Default declarative transaction behavior depends on a Spring-managed proxy intercepting the invocation. Calling a manually constructed object does not pass through that proxy. The annotation alone does not alter the behavior of every Java method call.

## U4 · REST design

**U4.1** Use 400 for input that violates the request contract, such as a nonpositive quantity. Use 409 when valid input conflicts with existing state, such as cancelling a reconciled order or reusing an idempotency key with different data. Document and test the exact choices.

**U4.2** An entity exposes persistence structure, may trigger lazy loads or recursive serialization, and can accidentally reveal internal fields. A DTO defines an intentional API contract and allows schema/persistence changes without automatically changing external responses.

**U4.3** Without stable ordering, records can move or be repeated/omitted between pages, especially when sort values tie. Add a unique tie-breaker such as ID. Concurrent inserts can still shift offset pagination; keyset/cursor pagination is a useful next step for changing datasets.

## U5 · HTTP clients

**U5.1** No. The server could commit and the response could be lost or late. The caller has an unknown outcome. Use a persistent idempotency key, status lookup, or reconciliation protocol rather than assuming no effect and submitting a new independent order.

**U5.2** The remote service can delay while the transaction retains locks and a database connection, increasing contention and exhausting pools. A database rollback cannot undo an already-completed HTTP side effect. Separate remote work from short DB transactions and define the consistency protocol.

**U5.3** Selected transient network failures, throttling, or temporary server failures may justify retries for read operations or writes with proven idempotency. Bound attempts, total elapsed time, and concurrent calls; apply backoff/jitter and honor relevant server guidance. Do not retry invalid input or authorization failures blindly.

## U6 · SQL

**U6.1** A sequential scan can cost less for a small table or a query returning much of it. Estimates, statistics, predicate expressions, column order, and sorting requirements also matter. Inspect actual plans and data distribution; an unused index is not automatically a planner defect.

**U6.2** Positive order quantity is one example. Java validation gives an early, useful API error; the database constraint protects every writer, including scripts or a future service. A unique account-scoped idempotency key is another invariant that must survive concurrent application instances.

**U6.3** An index consumes storage and adds work to inserts, updates, and maintenance. It may duplicate an existing index or benefit a rarely used query. Evaluate the workload's read/write mix and measured improvement before adding it.

## U7 · Transactions and persistence

**U7.1** A self-call stays within the object and does not enter through its Spring proxy. It therefore does not apply the method's intended new transaction advice in default proxy mode. Move the transactional boundary to an externally invoked service method or use intentional programmatic transaction management.

**U7.2** Optimistic versioning detects stale updates to that versioned entity and prevents silent lost updates on that row. It does not enforce all cross-row invariants, coordinate external HTTP work, or ensure duplicate request suppression. Those require additional constraints or protocols.

**U7.3** Only participating database work rolls back. A remote side effect might already have succeeded, or the database might commit while notification fails. Use an outbox for durable notification intent, idempotent consumers, and recovery/reconciliation for effects across systems.

## U8 · Security

**U8.1** Authentication establishes identity. A scope such as `orders.write` permits a category of action. Ownership checks whether that identity may act on this particular order/account. All three matter: a valid write token from account A must not cancel account B's order.

**U8.2** A mock JWT often supplies an already-authenticated principal directly to the security context. That checks authorization wiring but bypasses token decoding/signature verification. Use controlled real signed tokens to test the decoder, issuer, audience, and expiry behavior.

**U8.3** The assumption breaks if credentials are sent automatically by a browser, such as session cookies or some other browser-managed authentication. Stateless server code alone does not eliminate CSRF risk. Reassess the protection when adding a browser UI or changing token storage/transport.

## U9 · Testing

**U9.1** A real PostgreSQL integration test should execute the conflicting writes and observe the database's behavior. For idempotency, also test concurrent requests or transactions. A mocked repository can only confirm what you told the mock to return.

**U9.2** Coverage counts executed code, not correct assertions or meaningful scenarios. Tests can execute a method without checking its outcome, omit concurrency/security paths, or mirror an implementation's bug. Prefer invariant, failure, and boundary tests, and confirm that realistic defects make them fail.

**U9.3** The test transaction can keep a persistence context alive beyond the real application's boundary, hiding lazy-loading problems. It can also roll back before commit-time behavior is exercised. Test actual service transactions and explicitly flush/commit where the scenario requires it; use separate transactions for races.

## U10 · JVM and performance

**U10.1** Requests can spend most of their time waiting for database connections, locks, disk, remote services, or scheduling. Low CPU does not mean the service has spare usable capacity at its actual bottleneck. Correlate thread/trace and dependency-pool evidence.

**U10.2** The live `RequestHistory` object holds its list, and the list holds the arrays. They remain reachable, so GC must preserve them. Fix retention policy or references; switching collectors does not make live objects eligible for collection.

**U10.3** Keep dataset, workload mix, arrival rate, warm-up, hardware/resource limits, dependencies, and measurement windows comparable. Record throughput and error rate alongside latency so “faster” does not merely mean more rejected work. Repeat runs and report variability, not one favorable number.

## U11 · Concurrency

**U11.1** Increment is a read-modify-write sequence. Two threads can read the same value and both store the same incremented value. `volatile` provides visibility/order guarantees for accesses, not atomicity of that compound operation. Use an atomic operation or a lock appropriate to the invariant.

**U11.2** Throughput may be limited by CPU, database connections, a vendor quota, serialization, or a lock. Virtual threads mainly reduce the cost of having many tasks wait for I/O. They do not remove downstream capacity limits or make a sequential dependency parallel.

**U11.3** The semaphore is local to one process. Two replicas with 20 permits each can admit 40 calls in total. A global vendor limit needs a deliberate cross-instance budget/rate-limiting strategy or conservatively divided capacity, plus monitoring of actual fleet behavior.

## U12 · Clean code and patterns

**U12.1** Strategy helps when there are multiple interchangeable policies with a common contract, such as market-specific cancellation rules. For one stable rule, a clearly named method is usually enough. Extra abstraction must repay its cost in actual variation, testing, or clarity.

**U12.2** Refactoring preserves agreed observable behavior while changing internal structure. A behavior change alters outputs, failure handling, timing guarantees, or contracts. Separate them when possible, and use characterization/regression tests to make the boundary visible.

**U12.3** An adapter translates vendor-specific requests, responses, and failures into your application's vocabulary. It limits coupling and lets tests replace the remote boundary. It should not hide important distinctions such as “not found,” “unavailable,” and “invalid response” behind a single fake success.

## U13 · System design

**U13.1** Split when there is evidence of independent ownership, deployment cadence, scaling, isolation, or data-boundary needs that outweigh added operational/distributed-consistency cost. “Microservices are senior” is not a reason. Explain what changes in failure handling, deployment, and contracts after the split.

**U13.2** Compare the persisted normalized request fingerprint under the same authenticated account and key. If it matches, return the retained result; if it differs, reject with a documented conflict. The key alone cannot tell whether the caller repeated the same operation.

**U13.3** The client may retry although the first order exists. Without durable idempotency, it can create another order. Reusing the same key lets the service recover the original result. The user-facing outcome should acknowledge uncertainty until it is resolved.

## U14 · Messaging

**U14.1** It closes the gap between committing a business change and durably recording that an event must be published: both are stored in one DB transaction. It does not atomically combine broker publication and the later “sent” mark, nor does it eliminate all possible event-ordering issues.

**U14.2** The relay may publish successfully and crash before marking the row sent, or the consumer may commit its DB change and crash before acknowledging. Reprocessing is then expected. A stable event ID plus transactional deduplication makes duplicate delivery harmless for the defined DB effect.

**U14.3** They must commit in the same database transaction for this project. If the marker commits first and the effect fails, replay could wrongly skip missing work. Acknowledge/commit the consumed position after successful DB completion. External effects require their own idempotency/recovery protocol.

## U15 · Caching and reliability

**U15.1** No. The annotation chooses cache behavior and keys; TTL comes from the cache/provider configuration. Also distinguish expiration from active invalidation, and verify the actual provider's synchronization behavior rather than assuming a cross-replica lock.

**U15.2** A timeout limits how long one operation waits. A circuit breaker changes whether subsequent calls are attempted after observed failures, then tests for recovery. You may need both, plus concurrency limits; neither alone defines safe retry semantics.

**U15.3** Retries increase demand on an already struggling dependency. Multiple layers can multiply attempts, while synchronized retries cause spikes. Retry only eligible operations within a bounded deadline and attempt budget, with backoff/jitter and capacity limits. Never turn a failed write into multiple business effects.

## U16 · Docker and CI/CD

**U16.1** An image is a packaged filesystem/configuration template. A container is a running or stopped instance with runtime state and configured resources. Replacing the application container must not delete durable database state; use appropriate persistent storage and tested backups.

**U16.2** The old application may not understand a renamed/dropped column or changed data format. Rolling back only its image can leave it incompatible with the database. Prefer additive expand/contract migrations and remove old structures only after the compatibility window is intentionally closed.

**U16.3** A laptop can contain undocumented JDKs, settings, services, credentials, or stale artifacts. A fresh-checkout build, CI run, pinned artifact, documented configuration, smoke check, and recovery rehearsal show that another environment can reproduce the service.

## U17 · Cloud and observability

**U17.1** Metrics show when/where latency and error rates changed; traces show the time spent in each request/dependency segment; logs provide event details and context for a particular failure. Correlation links them. None substitutes for a clear hypothesis and a verified fix.

**U17.2** Every distinct ID can create a new time series, increasing storage, memory, and query cost dramatically. Use bounded labels such as operation and outcome; put identifiers in suitably protected logs/traces when needed for diagnosis.

**U17.3** Restarting every instance will not repair a shared failed database and can amplify load during recovery. Liveness should signal a process that needs restarting. Readiness and explicit degraded behavior handle inability to serve traffic, based on the service's real dependencies and useful work.

## U18 · Senior responsibilities

**U18.1** It records the original constraints, alternatives, decision, consequences, and a trigger for revisiting it. A later engineer can understand why the choice was reasonable then and whether the assumptions still hold. Recording only “we chose Kafka” is not enough.

**U18.2** Race-safe uniqueness, account scoping, and matching the request to the retained result are correctness/security requirements, so they are blocking. Naming or formatting preferences can be optional unless they materially obscure behavior or violate an agreed rule. State the risk and a testable remedy.

**U18.3** Describe specific actions and outcomes: clarified a design, improved a review, wrote a runbook others used, helped a colleague understand a failure mode, or coordinated mitigation. Distinguish your contribution from the team's. Use real evidence and numbers only when you have them; influence does not require a manager title.

## Final mixed assessment

Use 60 minutes on 28 December: 15 minutes for six previously missed questions, 20 minutes for a small Java/API change with a test, 15 minutes to explain a duplicate-event incident, and 10 minutes to defend an architectural choice.

Passing evidence: at least five of the six questions fully correct; the change has an appropriate test and explained edge cases; the incident explanation separates cause, impact, mitigation, and prevention; the design answer names a real alternative and consequence. Record weak areas and use the remaining session time for correction. This complements the project rubric rather than replacing it.
