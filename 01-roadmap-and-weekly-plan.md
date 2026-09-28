# Your roadmap and weekly plan

## What gets priority

Your Java 8 confidence is 3/3, API confidence is 1/3. The other areas are **unassessed**, rather than automatically beginner-level: production support, migrations, and query optimization already give you relevant experience. Week 1 will separate unfamiliar terminology from missing practical skills.

| Priority | Focus | Why and depth by December |
|---|---|---|
| 1 | Build and consume REST APIs; Spring Boot; automated testing; security | Your explicit gap and the basis of the entire project. Implement independently and defend failure behavior. |
| 2 | Modern Java, concurrency, transactions, JPA correctness, JVM diagnosis | Connect existing experience to modern tools and demonstrate correctness under concurrent use. |
| 3 | System design, distributed failures, messaging, caching | Explain tradeoffs and build one reliable asynchronous workflow. Avoid collecting frameworks. |
| 4 | Docker, CI/CD, deployment, observability | Reproduce a release, diagnose it, and recover from a failed change. |
| Every week | Technical decisions, reviews, mentoring, communication | Turn implementation and your production experience into senior-level evidence. |
| Short diagnostic review | Java syntax, basic collections, CRUD SQL, legacy Spring configuration | Increase time only if the diagnostic exposes a real gap. |

Core commitments: one service, PostgreSQL, one HTTP integration, security, tests, a container, CI, useful telemetry, and recoverable asynchronous processing. Kafka and Redis are small, bounded labs added to that service. Kubernetes, multiple cloud providers, reactive programming, native images, multi-region failover, and advanced GC tuning are optional after the core gates pass.

## Your repeating two-hour sessions

| Day | Exactly 120 minutes |
|---|---|
| Monday | 15 min recall from last week; 45 min assigned reading; 50 min small implementation; 10 min notes. |
| Wednesday | 15 min closed-book recall; 25 min reading; 70 min coding; 10 min notes. |
| Friday | 20 min revisit material from 1–3 weeks ago; 40 min tests/debugging; 40 min implementation; 20 min explain a tradeoff aloud. |
| Saturday | 10 min define acceptance checks; 95 min project work; 15 min save evidence. |
| Sunday | 30 min self-test; 30 min review incorrect answers; 30 min interview/leadership practice; 20 min adjust next week; 10 min log progress. |

Stop when the session ends. A failing test and a written next step are a legitimate stopping point. Review is already budgeted; do not add it on top of the ten hours. Within the interview slot, alternate Java coding, system design, and workplace examples. For a coding-heavy target role, replace 30 minutes of Saturday work every other week with a timed map/queue/string exercise and explain its complexity.

## Dated weekly plan

Unit numbers refer to the study notes. Each cell is the focus for the session template above. Gates are defined in the project guide; do not pass a gate just because the calendar advances.

| Week and dates | Monday | Wednesday | Friday | Saturday | Sunday evidence / gate |
|---|---|---|---|---|---|
| **1 · Sep 21–27** | Run baseline diagnostic; select JDK and create study repo | U1–2: record DTO, collections, generics; configure Maven Wrapper | U3/U9 basics: launch Boot app and write first domain test | Define order states and API contract; build one in-memory POST/GET path | Explain request → controller → service; baseline saved; **M0** |
| **2 · Sep 28–Oct 4** | U4: resources, status codes, validation | Build create/get/list endpoints; cap page size | Test invalid JSON, missing fields, missing IDs; ProblemDetail responses | Replace throwaway behavior with explicit service/domain boundaries | Demonstrate five API scenarios without a tutorial; API portion of **M1** |
| **3 · Oct 5–11** | U5: build an HTTP client; connect/read deadlines | Add mock instrument-information service and call it | Test 404, 500, invalid JSON, timeout, and empty body | Finish API/client vertical slice; no remote I/O inside a DB transaction | Explain what a timeout does and does not prove; **M1** |
| **4 · Oct 12–18** | U6: SQL diagnostic, schema, constraints, migration | U7: JPA mapping, explicit DTOs, persistence boundary | PostgreSQL integration tests; demonstrate and fix an N+1 query | Persist orders; add optimistic locking and transactional audit entry | Force rollback and stale update; **M2** |
| **5 · Oct 19–25** | U8: authentication, scopes, ownership | JWT resource server; read/write permissions | Test missing token, wrong scope, other account, expired token | Add ownership checks at service boundary; safe error responses | Explain issuer/audience validation and CSRF assumptions; **M3** |
| **6 · Oct 26–Nov 1** | U9: unit, HTTP, repository, integration test boundaries | U12: refactor cancellation policy under tests | Reproduce one defect and write regression test; review a diff | Consolidate M1–M3; remove fragile mocks; document test commands | First 30-min mock interview; **M4**; use this week for gaps before new infrastructure |
| **7 · Nov 2–8** | U11: races, visibility, atomic operations | Implement duplicate-request protection using PostgreSQL | Concurrent test: same idempotency key; stale version conflicts | Virtual-thread lab with bounded dependency capacity | Explain why a Java lock cannot protect two replicas; **M5** |
| **8 · Nov 9–15** | U10: JVM, heap, threads, JIT, JFR | Seed data and establish repeatable load baseline | Diagnose slow endpoint with SQL plan, pool metrics, and JFR | Fix one measured bottleneck and repeat identical workload | Performance report with before/after evidence; **M6** |
| **9 · Nov 16–22** | U13: requirements, consistency, capacity estimates | U14: transactional outbox; single relay first | Crash between publish and acknowledgment; test duplicates | Kafka lab: one topic and reconciliation consumer; DB deduplication | Draw failure timeline and explain recovery; **M7** core |
| **10 · Nov 23–29** | U15: cache-aside, TTL, staleness | Cache reference data; configure TTL; stop Redis | Retry/deadline drill; test overload and broker outage | Complete replay/reconciliation checks; integrate M7 and M8 | Explain stale-data and degraded-service decisions; **M7–M8** |
| **11 · Nov 30–Dec 6** | U16: image, process, ports, volumes | Docker Compose for service + database; add optional dependencies by profile | CI executes unit + PostgreSQL integration tests | Build tagged image; fresh-clone run; rehearse compatible DB migration | Reproducible release evidence; **M9** |
| **12 · Dec 7–13** | U17: metrics, logs, traces; implement useful telemetry | Sandbox cloud deployment, or labeled local rehearsal if no budget/account | Verify secret/config handling, health, timeouts, graceful stop | Smoke test deployed environment; rollback to previous compatible image | Deployment and recovery record; **M10**, with cloud status stated honestly |
| **13 · Dec 14–20** | U18: write ADR comparing two designs | Review PR for ownership/idempotency defects; revise review comments | Teach one concept; strengthen weakest diagnostic area | Simulated dependency incident: detect, mitigate, verify, write postmortem | 30-min system design mock; **M11**; draft promotion evidence |
| **14 · Dec 21–27** | Closed-book Java/API test; fix largest remaining gap | Fresh-run project demonstration; clean README | Timed debugging/coding mock, then review mistakes | Final recovery drill and portfolio evidence; no new features | Final rubric, interview answers, next actions; **M12** |
| **Final · Dec 28 & 30** | Dec 28: 60-min mixed assessment + 60-min targeted correction | Dec 30: 60-min demo + 30-min career evidence + 30-min next-month plan | No required session after Dec 30 | — | **4 hours**, not a full additional week |

Full weeks: 14 × 10 = 140 hours. Final sessions: 4 hours. Total: 144. If a holiday removes a session, record the lost capacity and reduce scope; do not assume make-up hours.

## Week 1 diagnostic: 45 minutes, inside Monday's session

Use notes only after attempting each item. Record 0 = cannot start, 1 = with guidance, 2 = independent, 3 = can test and explain tradeoffs.

1. **8 min, Java:** create an immutable order snapshot and group quantities by symbol; explain mutable map keys.
2. **8 min, APIs:** sketch POST/GET contracts and a Java client call; explain 400/401/403/404/409/503.
3. **7 min, data:** explain two simultaneous cancellations and where a transaction begins/ends.
4. **7 min, testing:** write one behavioral test and name what still requires a real database.
5. **5 min, runtime:** say what evidence distinguishes CPU, DB waiting, and memory retention.
6. **5 min, operations:** describe how to deploy a new version and recover when it fails.
7. **5 min, leadership:** give a real example of a tradeoff you influenced and how you measured the outcome.

Use the remaining first-session time to write the baseline and start setup. If environment setup is slow, use Wednesday's coding block; do not spend a whole week comparing IDEs or build tools.

## Adjusting the plan based on evidence

Each Sunday, choose three questions from that week's units, prioritizing your weakest unit, and score them **0–2 each**: 0 incorrect, 1 partly right, 2 correct with a concrete example or failure mode. Attempt the remaining questions in the next study day's recall block. Score the practical gate separately as pass, partial, or fail. Consult the answer key only after answering.

| Result | Change to next week within the same ten hours |
|---|---|
| 5–6/6 and gate passes independently | Continue. If this happens for two weeks, replace 60 min of familiar reading with a harder failure drill or mock interview. |
| 3–4/6 or gate is partial | Move 90 min of next week's new-feature work to the missed concept plus a fresh exercise. |
| 0–2/6 or gate fails | Use Monday and Wednesday, four hours total, to repair the prerequisite. Defer an optional lab; keep Sunday review. |
| More than two sessions lost | Re-estimate remaining hours. Preserve APIs, data correctness, security, testing, basic deployment, and recovery; shrink breadth. |
| Two failed attempts on the same issue | Write the smallest reproducible case and ask for targeted feedback with code, expected result, actual result, and logs. |

**Examples:** If Week 4 SQL is already strong but transaction tests fail, shorten SQL revision and spend the recovered time on concurrent updates. If Week 5 authorization is weak, use Week 6 to repair it before concurrency or messaging. If Week 9 outbox work overruns, retain the durable outbox and failure explanation; reduce Kafka tooling exploration and the Redis lab. Record any unfinished acceptance check as deferred, not passed.

**Spaced review:** revisit each concept on the next study day, about a week later, and three weeks later. Use Friday's 20 minutes for older units and Sunday for current ones. Re-answer a missed question in your own words, then change the example so you cannot pass by memorization.

This pack is ready to adapt when you provide a weekly check-in using the tracker. No recurring task or automatic progress monitoring has been set up.

## Three career outputs

- **Interviews:** demonstrate the project in ten minutes; explain a tradeoff in two minutes; solve one bounded Java problem and test edge cases; give three honest production stories.
- **Promotion:** map evidence to your employer's actual expectations. Show ownership, reduced recurring toil, better incident response, reviews, and team enablement. Ask for a stretch responsibility, then record its real outcome.
- **New job:** present a runnable repository and architecture/failure notes. Translate existing production work into impact statements with real numbers where available. Do not invent metrics or imply the study project served actual trading traffic.
