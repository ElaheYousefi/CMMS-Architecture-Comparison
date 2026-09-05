# CMMS Architecture: Three Implementations, Three Lessons

I built the same Computerized Maintenance Management System three times, using three different architectural patterns — not to find a "winner," but to understand when and why you'd choose each one, and how to evolve between them as scale and team size grow.

Each version implements the same workflow: define a maintenance task → schedule it → detect it's ready → generate a work order → assign a technician → record results → update equipment status.

| | [Monolithic](https://github.com/ElaheYousefi/LayeredCMMS) | [Event-Driven Modular ⭐](https://github.com/ElaheYousefi/EventDrivenModularCMMS) | [Microservices](https://github.com/ElaheYousefi/CMMS-Microservices) |
|---|---|---|---|
| Codebase | Single | Single | Multiple |
| Database | 1 shared | 1 shared | 1 per service |
| Coupling | Tight (direct calls) | Loose (events) | Loose (Kafka) |
| Consistency | ACID | ACID + eventual | Eventual |
| API response time | 50–100ms | ~50ms | 100–200ms |
| Async support | Limited | Full (`@Async`) | Full (Kafka) |
| Event durability | N/A | In-memory (lost on crash) | Kafka-backed (survives crash) |
| Debugging | Easy | Easy–medium | Hard (distributed) |
| Write throughput | < 1,000/sec | < 5,000/sec | > 10,000/sec |
| Team size | 1–10 | 5–15 | 15+ |
| Operational load | Low | Low–medium | High |

---

## 1. Monolithic Layered

**Stack:** Java, Spring Boot, PostgreSQL, Flyway, JWT, Docker

Controller → Service → Repository, all against a single database, wrapped in `@Transactional` for atomic, ACID-consistent writes.

**Strengths:** dead-simple to debug (one stack trace, one codebase), one deployment, a new engineer can understand the whole system in a week.

**Breaks down around:** 10M+ records in a hot table, 1,000+ writes/sec (lock contention under ACID), or a team bigger than ~10 (one bug can take down every module).

**Use it** for small teams and data volumes where strong consistency matters more than raw throughput. **Avoid it** once features need to scale or deploy independently.

---

## 2. Event-Driven Modular ⭐ (my recommendation)

**Stack:** Java 11+, Spring Boot, PostgreSQL, Spring `ApplicationEventPublisher`, `@EventListener` + `@Async`, Docker

Same single database as the monolith, but modules communicate through published events instead of direct calls — so Equipment doesn't need to know Notification or Dashboard exist.

The real payoff is moving slow, non-critical work off the request thread:

```
Blocking:      request → send email (500ms) → send SMS (100ms) → response   = 600ms
Async: request → queue event → response (50ms)   [email/SMS run in background]
```

That's a 12x drop in API response time, achieved with `@Async` alone — no new infrastructure.

**Strengths:** loose coupling without distributed-systems complexity, fast responses, easy to test (mock the listeners), easy to extend (new feature = new `@EventListener`, zero changes to existing code).

**Breaks down around:** the same single-database/single-machine ceiling as the monolith, plus in-memory events are lost if the JVM crashes before they're processed (fixable with `@Transactional` + an outbox table).

**Use it** as the default for most production systems — it's the sweet spot between monolith simplicity and microservices complexity, and it leaves you a clean migration path if you outgrow it. **Avoid it** only if you're a team of 1–4 (unnecessary complexity) or need guaranteed event delivery today.

---

## 3. Microservices with Kafka

**Stack:** Java 21, Spring Boot 3.x, PostgreSQL per service, Apache Kafka, Transactional Outbox Pattern, Docker Compose

Each service (Equipment, Maintenance, WorkOrder, Notification) owns its own database and reacts to events on Kafka instead of calling other services directly. The **Transactional Outbox Pattern** is what makes this reliable: a service writes its state change and an outbox record in the same local transaction, and a separate process publishes the outbox to Kafka — so an event is never lost even if the process crashes between the write and the publish.

**Strengths:** each service scales and deploys independently (Equipment can handle 10,000 events/sec while WorkOrder handles 100), events persist through crashes, teams can work independently.

**Breaks down around:** everything gets harder — eventual consistency surprises, cross-service debugging, 50–100ms of network latency per hop, and real operational overhead (4 services, 4 databases, Kafka, all needing monitoring).

**Use it** when a team exceeds ~15 engineers, needs independent deploy cycles, or data volume exceeds what one database can hold. **Avoid it** below that — the operational cost isn't worth it yet.

---

## What I Learned

1. **Start simple.** The monolith was the fastest to build and the easiest to reason about — and for small teams and data, it's genuinely the right answer, not just a starting point.
2. **Find the real bottleneck before adding architecture.** My monolith's problem wasn't "not enough microservices" — it was notifications blocking the request thread. `@Async` fixed it completely.
3. **Loose coupling + async + a single database is usually enough.** It gets you most of what microservices promise (extensibility, fast responses, independent-feeling modules) without distributed-systems cost. This is where I'd start almost any new production system today.
4. **Only go distributed when you've actually hit a wall** — sustained writes above ~5,000/sec, teams needing independent deploys, or data too large for one machine. Microservices adopted early are a cost with no matching benefit.
5. **Every choice is a trade-off, not a verdict.** ACID vs. eventual consistency, coupling vs. independence, simplicity vs. scale — the right answer depends on your constraints (team size, throughput, consistency needs), not on which architecture sounds most impressive.

**Evolution path, if scale eventually demands it:** monolith → add async processing → add an outbox table for durability → swap `ApplicationEventPublisher` for Kafka → decompose into services. Each step is additive, not a rewrite.
