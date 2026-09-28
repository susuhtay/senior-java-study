# Senior Java Developer study pack

Prepared for Java Developer · 19 September 2026 · Target: before January 2027

11 years of Java 8, Spring, Struts 2, Hibernate/JPA, SQL, and production-support experience in enterprise financial systems. This plan builds on that foundation. Your clearest stated gap is both building and consuming APIs; you also want confidence across modern backend engineering and senior responsibilities.

**Schedule:** Monday, Wednesday, Friday, Saturday, Sunday; two hours each day. Start Monday 21 September. There are 14 full study weeks through 27 December, then two final sessions on 28 and 30 December: **144 available hours**. Tuesday and Thursday are free. No extra homework is assumed.

The outcome is evidence that you can design, implement, test, operate, and explain a backend service. Completing a syllabus alone does not establish seniority; the project gates and your existing workplace examples are the evidence to use in interviews and promotion discussions.

## Your materials

1. [Roadmap and weekly plan](01-roadmap-and-weekly-plan.md) — priorities, dated sessions, review routine, and adjustment rules.
2. [Study notes and exercises](02-study-notes.md) — 18 learning units with explanations, Java examples, production mistakes, exercises, and 54 self-test questions.
3. [Progressive project](03-project-and-assessment.md) — an order-processing and reconciliation service, milestones, acceptance criteria, and failure drills.
4. [Separate answer key](04-answer-key.md) — read only after attempting the questions.
5. [Progress tracker](05-progress-tracker.md) — baseline, weekly check-in, evidence log, and readiness rubric.

Start with the first Monday session in the weekly plan. Read only the units assigned that week; this pack is a working reference, not a book to finish before coding.

## Technical baseline

Use **JDK 25, Spring Boot 4.1.x, Maven Wrapper, PostgreSQL 18, and Git** for the new practice service. The official Boot requirements page currently identifies 4.1.1 and supports Java 25; use its managed dependency versions instead of choosing independent Spring, Hibernate, and JUnit versions. Java 25 is an LTS release. The examples focus on stable features available in Java 21 and later, without preview features. [Spring Boot requirements](https://docs.spring.io/spring-boot/system-requirements.html), [Java support roadmap](https://www.oracle.com/java/technologies/java-se-support-roadmap.html).

Pin the actual versions generated in Week 1 in the project README. Do not chase new releases every week. If a target employer uses Boot 3 or Java 17/21, add a short comparison note and use the matching version of its documentation. The goal is transferable understanding, not memorizing the newest APIs.

The local Java command currently reports JDK 23.0.1. Week 1 includes selecting a JDK 25 installation for this study project; no runtime or dependencies have been changed by creating this pack.

Learn to recognize the migration from enterprise `javax.persistence` / `javax.validation` packages to `jakarta.*`; **do not** mechanically replace every `javax` package, because Java SE still has packages such as `javax.sql`. Treat upgrading an existing Java 8 application as a separate, tested migration, not as a compiler-setting change.

Assume local development first and no required paid courses. Docker availability and cloud budget are unconfirmed. Week 1 checks Docker; Week 12 offers a real sandbox deployment or a clearly labeled local deployment rehearsal. Rehearsal is useful but does not count as completed cloud deployment.

## Reading and example conventions

Sources are official project, vendor, or engineering-team documentation checked on 19 September 2026. Live documentation can change; use the version selector matching your build. Read the named section, not every page of an entire manual.

Java snippets are focused teaching excerpts, not a complete application. Unless explicitly included, imports, constructors, dependency injection, and supporting domain types are omitted. Names such as `OrderService`, `OrderView`, and `OrderNotFound` describe types you will implement in the project. Framework examples must be compiled and exercised in that project; this pack does not claim they have passed an application build.

Project limits, workload sizes, latency targets, and grading thresholds are proposed learning criteria, not published industry standards or claims about your employer's systems. Use synthetic data and your own designs for the portfolio.
