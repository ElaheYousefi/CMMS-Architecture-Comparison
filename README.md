# CMMS Architecture: Three Implementations, Three Lessons

I built the same Computerized Maintenance Management System three times, using three different architectural patterns — not to find a "winner," but to understand when and why you'd choose each one, and how to evolve between them as scale and team size grow.

Each version implements the same workflow: define a maintenance task → schedule it → detect it's ready → generate a work order → assign a technician → record results → update equipment status.

> **Note on the figures:** The response-time, throughput and team-size values in the table below are rough estimates based on the typical behavior of each pattern. They are **not** benchmark results from this code. The only numbers tied to this code are the simulated delays described in section 2.

|                                 | [Monolithic](https://github.com/ElaheYousefi/LayeredCMMS) | [Event-Driven Modular ⭐](https://github.com/ElaheYousefi/EventDrivenModularCMMS) | [Microservices](https://github.com/ElaheYousefi/CMMS-Microservices) |
| ------------------------------- | --------------------------------------------------------- | -------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| Codebase                        | Single                                                    | Single                                                                           | Multiple                                                            |
| Database                        | 1 shared                                                  | 1 shared                                                                         | 1 per service                                                       |
| Coupling                        | Tight (direct calls)                                      | Loose (events)                                                                   | Loose (Kafka)                                                       |
| Consistency                     | ACID                                                      | ACID + eventual                                                                  | Eventual                                                            |
| API response time (estimated)   | 50–100ms                                                  | ~50ms                                                                            | 100–200ms                                                           |
| Async support                   | Limited                                                   | Full (`@Async`)                                                                  | Full (Kafka)                                                        |
| Event durability                | N/A                                                       | In-memory (lost on crash)                                                        | Kafka-backed (survives crash)                                       |
| Debugging                       | Easy                                                      | Easy–medium                                                                      | Hard (distributed)                                                  |
| Write throughput (estimated)    | < 1,000/sec                                               | < 5,000/sec                                                                      | > 10,000/sec                                                        |
| Team size (rule of thumb)       | 1–10                                                      | 5–15                                                                             | 15+                                                                 |
| Operational load                | Low                                                       | Low–medium                                                                       | High                                                                |

---

## 1. Monolithic Layered

**Stack:** Java, Spring Boot, PostgreSQL, Flyway, JWT, Docker

Controller → Service → Repository, all against a single database, wrapped in `@Transactional` for atomic, ACID-consistent writes.

**Strengths:** dead-simple to debug (one stack trace, one codebase), one deployment, a new engineer can understand the whole system in a week.

**Typically breaks down around** (rule of thumb, not measured here): 10M+ records in a hot table, 1,000+ writes/sec (lock contention under ACID), or a team bigger than ~10 (one bug can take down every module).

**Use it** for small teams and data volumes where strong consistency matters more than raw throughput. **Avoid it** once features need to scale or deploy independently.

---

## 2. Event-Driven Modular ⭐ (my starting point for systems of this size)

**Stack:** Java 11+, Spring Boot, PostgreSQL, Spring `ApplicationEventPublisher`, `@EventListener` + `@Async`, Docker

Same single database as the monolith, but modules communicate through published events instead of direct calls — so Equipment doesn't need to know Notification or Dashboard exist.

The real payoff is moving slow, non-critical work off the request thread:

```
Blocking:      request → send email (500ms) → send SMS (100ms) → response   = 600ms
Async:         request → queue event → response (50ms)   [email/SMS run in background]
```

In this demo, the email and SMS calls are **simulated** with delays of 500 ms and 100 ms. A blocking request therefore takes about 600 ms, while the `@Async` version responds in about 50 ms and runs the notifications in the background. That is roughly a 12x drop in this simulated scenario, achieved with `@Async` alone — no new infrastructure. It illustrates the effect of moving slow work off the request thread; it is not a production benchmark.

**Strengths:** loose coupling without distributed-systems complexity, fast responses, easy to test (mock the listeners), easy to extend (new feature = new `@EventListener`, zero changes to existing code).

**Typically breaks down around:** the same single-database/single-machine ceiling as the monolith, plus in-memory events are lost if the JVM crashes before they're processed (fixable with `@Transactional` + an outbox table).

**Use it** as my starting point for most production systems of this size — it's the sweet spot between monolith simplicity and microservices complexity, and it leaves a clean migration path if you outgrow it. **Avoid it** only if you're a team of 1–4 (unnecessary complexity) or need guaranteed event delivery today.

---

## 3. Microservices with Kafka

**Stack:** Java 21, Spring Boot 3.x, PostgreSQL per service, Apache Kafka, Transactional Outbox Pattern, Docker Compose

Each service (Equipment, Maintenance, WorkOrder, Notification) owns its own database and reacts to events on Kafka instead of calling other services directly. The **Transactional Outbox Pattern** is what makes this reliable: a service writes its state change and an outbox record in the same local transaction, and a separate process publishes the outbox to Kafka — so an event is never lost even if the process crashes between the write and the publish.

**Strengths:** each service scales and deploys independently (for example, Equipment could handle a high event rate while WorkOrder handles far less), events persist through crashes, teams can work independently.

**Typically breaks down around:** everything gets harder — eventual consistency surprises, cross-service debugging, added network latency on every hop, and real operational overhead (4 services, 4 databases, Kafka, all needing monitoring).

**Use it** when a team exceeds ~15 engineers, needs independent deploy cycles, or data volume exceeds what one database can hold. **Avoid it** below that — the operational cost isn't worth it yet.

---

## What I Learned

1. **Start simple.** The monolith was the fastest to build and the easiest to reason about — and for small teams and data, it's genuinely the right answer, not just a starting point.
2. **Find the real bottleneck before adding architecture.** My monolith's problem wasn't "not enough microservices" — it was notifications blocking the request thread. In my demo, `@Async` took that work off the request path without any new infrastructure.
3. **Loose coupling + async + a single database is often enough.** It gets you much of what microservices promise (extensibility, fast responses, independent-feeling modules) without distributed-systems cost. This is where I would start most new systems of this size.
4. **Only go distributed when you've actually hit a wall** — for example sustained high write rates, teams needing independent deploys, or data too large for one machine (the thresholds in this document are rules of thumb). Microservices adopted early are a cost with no matching benefit.
5. **Every choice is a trade-off, not a verdict.** ACID vs. eventual consistency, coupling vs. independence, simplicity vs. scale — the right answer depends on your constraints (team size, throughput, consistency needs), not on which architecture sounds most impressive.

**Evolution path, if scale eventually demands it:** monolith → add async processing → add an outbox table for durability → swap `ApplicationEventPublisher` for Kafka → decompose into services. Each step is additive, not a rewrite.

## Limitations

- The comparison table contains estimates and rules of thumb, not benchmark results.
- The 600 ms vs. 50 ms example uses simulated email and SMS delays, not real notification providers.
- All three implementations are simplified versions of a CMMS domain, built to compare architectures rather than to run in production.
