CMMS Architecture: Three Implementations, Three Lessons (UPDATED)

Portfolio Projects:

•	Monolithic: https://github.com/ElaheYousefi/LayeredCMMS

•	Event-Driven Modular: https://github.com/ElaheYousefi/EventDrivenModularCMMS ⭐ (Production-Ready)

•	Microservices: https://github.com/ElaheYousefi/CMMS-Microservices

\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

Executive Summary

I implemented the same Computerized Maintenance Management System (CMMS) using three fundamentally different architectural patterns. Not to find the "best" architecture, but to understand when and why you'd choose each one, and how to evolve between them.

Each implementation handles the same business workflow:

1\.	Define maintenance task

2\.	Schedule task execution

3\.	Detect task ready → generate work order

4\.	Assign technician

5\.	Record maintenance results

6\.	Update equipment status

Different architectures solve this differently. The choice depends on your constraints: team size, scale, consistency requirements, and performance needs.

\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

Architecture 1: Monolithic Layered

Repository: https://github.com/ElaheYousefi/LayeredCMMS

What It Is

Single codebase, single database, layered structure (Controller → Service → Repository).

Request

&#x20;  ↓

Controller Layer (HTTP handling)

&#x20;  ↓

Service Layer (business logic)

&#x20;  ↓

Repository Layer (database access)

&#x20;  ↓

PostgreSQL (single database)

Technology Stack

•	Java, Spring Boot

•	PostgreSQL (single instance)

•	Flyway (schema versioning)

•	JWT authentication

•	Swagger/OpenAPI docs

•	Docker (containerization)

•	JUnit 5, Mockito (testing)

Design Decisions I Made

1\. Single Database Design

•	All tables in one schema

•	Clear relationships between entities

•	ACID transactions for consistency

2\. ACID Transactions

java

@Transactional

public WorkOrder createWorkOrder(MaintenanceTask task) {

&#x20;   // All updates happen atomically

&#x20;   task.markReady();

&#x20;   workOrder.create(task);

&#x20;   equipment.updateLastMaintenance();

&#x20;   // If ANY fails, entire transaction rolls back

}

3\. Automated Scheduling

•	Scheduler runs every minute, checks for ready tasks

•	Automatically generates work orders

•	No manual intervention needed

Strengths

Strength	Benefit

ACID Consistency	Task creation + equipment status update happen together atomically

Simple Debugging	Stack trace shows entire flow, single codebase to understand

One Deployment	Deploy once, all features updated, no coordination

Single Database	No distributed transaction complexity

Easy to Understand	New engineer grasps entire system in 1 week

Weaknesses \& When It Fails

Problem	Scale	Why

Database bottleneck	10M+ maintenance records	Single database queries slow down as table grows

Concurrent write limits	1,000+ writes/second	ACID transactions cause lock contention

CPU bottleneck	10,000+ concurrent users	Single machine can't handle, vertical scaling has limits

Deployment risk	Team > 10 engineers	Bug in maintenance module affects equipment module

Trade-offs I Faced

Trade-off 1: Simplicity vs. Scaling

•	Chose: Simplicity (ACID, single DB)

•	Problem: At 1,000+ writes/second, lock contention degrades performance

•	Lesson: For < 1,000 writes/sec, monolith wins. For > 5,000, need sharding.

Trade-off 2: Consistency vs. Performance

•	Chose: Strong consistency (ACID)

•	Problem: Transaction locks slow concurrent writes

•	Lesson: ACID is expensive. Use where consistency matters. Loosen elsewhere.

Real-World Performance

•	API response time: 50-100ms (single database)

•	Write throughput: \~1,000 writes/second (single table)

•	Concurrent connections: \~5,000 (single machine limit)

•	Data consistency: Immediate (ACID guarantees)

•	Deployment time: 2 minutes (single JAR)

When to Use Monolithic

✅ Team < 10 engineers

✅ Data < 10M records (hot table)

✅ Writes < 1,000/second

✅ Tightly coupled features

✅ ACID consistency required

When NOT to Use Monolithic

❌ Team > 20 engineers

❌ Writes > 5,000/second

❌ Features scale independently

❌ Separate deployment cycles needed

\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

Architecture 2: Event-Driven Modular (PRODUCTION-READY) ⭐

Repository: https://github.com/ElaheYousefi/EventDrivenModularCMMS

What It Is

Single codebase with multiple loosely-coupled modules communicating via events. Spring's ApplicationEventPublisher enables modules to publish domain events that other modules listen to asynchronously.

Equipment Module  ──┐

&#x20;                    ├─ Single Database (PostgreSQL)

Maintenance Module ──├─ Spring ApplicationEventPublisher

&#x20;                    ├─ @EventListener @Async

Work Order Module ───┤

&#x20;                    ├─ Thread Pool for async processing

Notification Module──┘

Technology Stack

•	Java 11+, Spring Boot

•	PostgreSQL (single, shared database)

•	Spring ApplicationEventPublisher (in-memory event bus)

•	@EventListener with @Async (asynchronous processing)

•	@EnableAsync (thread pool configuration)

•	Docker (containerization)

•	JUnit 5, Testcontainers (testing)

Architecture Pattern

Event-Driven with Asynchronous Processing

java

// Equipment Module publishes event

@Service

public class EquipmentService {

&#x20;   @Autowired

&#x20;   private ApplicationEventPublisher eventPublisher;

&#x20;   

&#x20;   public void updateEquipmentStatus(String equipmentId, String status) {

&#x20;       equipment.setStatus(status);

&#x20;       equipmentRepo.save(equipment);

&#x20;       

&#x20;       // Publish event asynchronously

&#x20;       eventPublisher.publishEvent(

&#x20;           new EquipmentStatusChangedEvent(equipmentId, status)

&#x20;       );

&#x20;   }

}



// Notification Module listens and reacts asynchronously

@Service

public class NotificationService {

&#x20;   

&#x20;   @EventListener

&#x20;   @Async  // Runs in thread pool, doesn't block request

&#x20;   public void onEquipmentStatusChanged(EquipmentStatusChangedEvent event) {

&#x20;       sendEmail(event);

&#x20;       sendSMS(event);

&#x20;       // Completes in background, doesn't affect API response time

&#x20;   }

}



// Dashboard Module also listens independently

@Service

public class DashboardService {

&#x20;   

&#x20;   @EventListener

&#x20;   @Async

&#x20;   public void onEquipmentStatusChanged(EquipmentStatusChangedEvent event) {

&#x20;       updateDashboardWidget(event);

&#x20;   }

}

Key Design Decisions

1\. Loose Coupling via Events

Without events (tightly coupled):

├─ Equipment Service knows about Notification Service

├─ Equipment Service knows about Dashboard Service

└─ Adding new feature requires changing Equipment Service



With events (loosely coupled):

├─ Equipment Service publishes event

├─ Notification Service listens independently

├─ Dashboard Service listens independently

└─ Add new listener without touching Equipment Service

2\. Asynchronous Processing with @Async

Without @Async (blocking):

Request → publishEvent → sendEmail (500ms) → sendSMS (100ms) → Response

Total: 600ms ❌



With @Async (non-blocking):

Request → publishEvent → queue to thread pool → Response (50ms) ✅

\[Background] sendEmail (500ms) + sendSMS (100ms)

3\. Single Database (No Distributed Complexity)

•	All modules share one PostgreSQL

•	ACID transactions still available for critical operations

•	No eventual consistency issues

•	No distributed transaction complexity

Strengths

Strength	Benefit

Loose Coupling	Equipment Service doesn't know about Notification Service. Easy to extend.

Async Processing	Critical operations (work order) fast, non-critical (notifications) in background

Fast API Response	50ms response time vs. 600ms with blocking notifications

Easy to Test	Mock event listeners, test modules independently

Single Responsibility	Each module has one reason to change

Easy to Extend	Add new feature (ReportService) with @EventListener, no changes to existing code

Production-Ready	Single database + async = simple + performant

Weaknesses \& When It Fails

Problem	Scale	Why

Single database bottleneck	10M+ hot table records	Like monolithic, hits database limit at scale

Single machine limit	10,000+ concurrent users	Vertical scaling limits on single machine

Event processing bottleneck	100,000+ events/second	Thread pool capacity limits

No distributed reliability	Event loss on crash	Events in memory, lost if JVM crashes (solved with @Transactional + Outbox pattern)

Trade-offs I Made

Trade-off 1: Loose Coupling vs. Single Database

•	Chose: Loose coupling + single database

•	Benefit: Clean architecture + no distributed complexity

•	Limitation: Can't scale features independently

Trade-off 2: Asynchronous Performance vs. Immediate Notification

•	Chose: Async (user gets fast response, notifications 100-500ms later)

•	Benefit: 12x faster API response (600ms → 50ms)

•	Limitation: Notifications not guaranteed (if JVM crashes before send)

Trade-off 3: Simple Event Bus vs. Distributed Reliability

•	Chose: Simple in-memory events (ApplicationEventPublisher)

•	Benefit: No Kafka complexity, single database

•	Limitation: No event persistence, events lost on crash

Real-World Performance

•	API response time: 50ms (fast, doesn't wait for background tasks)

•	Event throughput: 10,000-50,000 events/second (thread pool dependent)

•	Data consistency: Immediate for critical ops, eventual for non-critical

•	Concurrent connections: \~5,000 (single machine, but async helps)

•	Deployment: Single JAR (simple), but loosely coupled (clean)

When to Use Event-Driven Modular

✅ Team 5-15 engineers

✅ Single database acceptable

✅ Need clean architecture (loose coupling)

✅ Performance matters (async processing)

✅ Want production-ready monolithic

✅ Future migration to microservices planned

When NOT to Use Event-Driven Modular

❌ Team < 5 (overkill complexity)

❌ Need guaranteed event delivery (use Outbox Pattern)

❌ Distributed teams need independent deployments

❌ Very high throughput (> 100,000 events/sec)

Why This Is My Recommendation

For most production monolithic systems, this is the sweet spot:

1\.	Better than monolithic - Loose coupling, easy to test, easy to extend

2\.	Simpler than microservices - Single database, no network complexity, easy to debug

3\.	Production-ready - Async processing, fast API responses, clean architecture

4\.	Evolutionary path - When scale forces it, migrate to microservices (loose coupling ready)

\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

Architecture 3: Microservices with Kafka

Repository: https://github.com/ElaheYousefi/CMMS-Microservices

What It Is

Multiple independent services, each owns its domain and database. Services communicate via events through Apache Kafka for distributed reliability.

Equipment Service ┐

&#x20;                 ├─ PostgreSQL (separate instance)

Maintenance Svc ──┤

&#x20;                 ├─ Apache Kafka (distributed event bus)

WorkOrder Service ├─

&#x20;                 ├─ PostgreSQL (separate instance)

Notification Svc──┘

Technology Stack

•	Java 21, Spring Boot 3.x

•	PostgreSQL (separate instance per service)

•	Apache Kafka (distributed event broker)

•	Transactional Outbox Pattern (reliable event publishing)

•	Docker Compose (local orchestration)

•	JUnit 5, Testcontainers (testing)

Service Decomposition

Service	Responsibility	Database	Scale Driver

Equipment Service	Equipment data, status, history	PostgreSQL	Sensor events (high volume)

Maintenance Service	Maintenance plans, scheduling	PostgreSQL	Task definitions (low volume)

WorkOrder Service	Work orders, technician assignment	PostgreSQL	Task readiness (medium)

Notification Service	Alert delivery (email, SMS)	No DB needed	Notification requests (async)

Event Flow

Maintenance Service → MaintenanceTaskReadyEvent → Kafka

&#x20;                                                    ↓

WorkOrder Service consumes → creates work order → WorkOrderCreatedEvent → Kafka

&#x20;                                                    ↓

Notification Service consumes → sends email/SMS

The Critical Pattern: Transactional Outbox

Ensures events and database stay consistent without distributed transactions:

java

@Transactional

public void publishEvent(Equipment equipment) {

&#x20;   // Both updates in SAME transaction

&#x20;   equipment.updateStatus("repaired");

&#x20;   outboxPublisher.insert(new EquipmentEvent());

&#x20;   // Commit atomically or rollback entirely

&#x20;   

&#x20;   // Separate thread publishes Outbox events to Kafka

&#x20;   // If crash between steps, Outbox events still exist and are published on recovery

}

Strengths

Strength	Benefit

Independent Scaling	Equipment (10,000 events/sec) scales separately from WorkOrder (100 events/sec)

Loose Coupling	Services don't call each other, only react to events

Team Independence	Equipment team deploys independently from WorkOrder team

Distributed Reliability	Events persisted in Kafka, survive crashes

Technology Flexibility	Equipment uses TimescaleDB, others use PostgreSQL

Weaknesses \& When It Fails

Problem	Scale	Why

Eventual Consistency	All services	Technician notification sent before UI updates

Debugging Complexity	Request flows across services	Hard to trace requests through Kafka

Network Latency	Every inter-service call	50-100ms per REST/Kafka call

Operational Overhead	4 services + Kafka + 4 databases	Complex monitoring, multiple failure modes

Testing Difficulty	Event ordering	Must test async behavior, failure scenarios

When to Use Microservices

✅ Team > 15 engineers

✅ Features scale independently

✅ Separate deployment cycles critical

✅ Data volume > 100M (multiple services)

✅ Distributed teams (time zones, geography)

When NOT to Use Microservices

❌ Team < 10 engineers

❌ Simple features

❌ Immediate consistency required (financial)

❌ High operational overhead burden

\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

Side-by-Side Comparison

Aspect	Monolithic	Event-Driven Modular ⭐	Microservices

Codebase	Single	Single	Multiple

Database	1 shared	1 shared	Many (one per service)

Module coupling	Tight (direct calls)	Loose (events)	Loose (Kafka)

Consistency	ACID	ACID + eventual	Eventual only

API response time	50-100ms (good)	50ms (fast)	100-200ms (network latency)

Async support	Limited	Full (@Async)	Full (Kafka)

Distributed reliability	N/A	Limited (in-memory)	High (Kafka)

Debugging	Easy	Medium (clean)	Hard (distributed)

Deployment	Single release	Single release	Independent releases

Testing	Hard (tight coupling)	Easy (mock listeners)	Very hard (distributed)

Scalability	Limited (single machine)	Limited (single machine)	High (distribute services)

Operational load	Low	Low-Medium	High

Learning curve	Low	Low-Medium	Very high

Team size	1-10	5-15	15+

Data volume	< 10M	< 100M	100M+

Write throughput	< 1,000/sec	< 5,000/sec	> 10,000/sec

\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

My Evolution as an Architect

Lesson 1: Start Simple

I built monolithic first. It was simple, fast to develop, easy to debug. For small teams and small data, it's perfect.

Lesson 2: Recognize the Bottleneck

After building monolithic, I identified the real problem: notifications were blocking requests. This isn't solved by more code, but by async processing.

Lesson 3: Event-Driven Modular Is the Sweet Spot

After implementing @Async, I realized: this is production-ready architecture. Loose coupling + async processing + single database = best of both worlds.

For most companies, Event-Driven Modular is where you should be.

Lesson 4: Know When to Evolve

Only evolve to microservices when you've hit the limits of single machine:

•	Writing > 5,000/sec to database

•	Need independent team deployments

•	Data is too large for one machine

Premature microservices is expensive.

Lesson 5: Architecture Is About Trade-offs

There is no "best" architecture. Choose based on:

•	Team size: < 10 → monolithic, 10-20 → event-driven, 20+ → microservices

•	Write throughput: < 1,000/sec → monolithic, 1-5,000 → event-driven, > 5,000 → microservices

•	Consistency: Financial → ACID, Notifications → eventual

•	Single point of failure: Critical? Keep monolithic. Resilience needed? Use Kafka.

\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

What I'd Choose Today (And Why)

For production CMMS: Event-Driven Modular with @Async

Structure:

Equipment Module (in-memory events)

&#x20; - Manages equipment data

&#x20; - Publishes EquipmentStatusChangedEvent

&#x20; 

Maintenance Module (in-memory events)

&#x20; - Task scheduling, detection

&#x20; - Publishes MaintenanceTaskReadyEvent



WorkOrder Module (in-memory events, @Async)

&#x20; - Creates work orders

&#x20; - Publishes WorkOrderAssignedEvent



Notification Module (@Async listener)

&#x20; - Listens to WorkOrderAssignedEvent

&#x20; - Sends email/SMS in background



Dashboard Module (@Async listener)

&#x20; - Listens to equipment events

&#x20; - Updates UI asynchronously

Why this choice:

•	✅ Clean architecture (loose coupling)

•	✅ Production performance (50ms API response)

•	✅ Simple operations (single database)

•	✅ Easy to test (mock event listeners)

•	✅ Migration path (if scale forces microservices)

What I'd avoid:

•	❌ Monolithic (harder to test, tight coupling)

•	❌ Full event sourcing (complexity without benefit)

•	❌ Premature microservices (operational overhead)

If scale demands (100M+ events/day):

•	Add Outbox Pattern for event persistence

•	Replace ApplicationEventPublisher with Kafka

•	Decompose into microservices

This is evolution, not revolution.

\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

Key Architectural Principles

1\. Understand the Bottleneck

•	Monolithic bottleneck: database locks on hot tables (solve with async)

•	Microservices bottleneck: network latency (solve with caching)

•	Event-driven bottleneck: event throughput (solve with Kafka)

2\. Trade-offs Matter

•	Loose coupling requires event handling (more code)

•	Async processing requires thread pool management (more monitoring)

•	Microservices require distributed tracing (more infrastructure)

Choose based on which trade-off fits your constraints.

3\. Consistency Is Expensive

•	ACID is expensive (locks, transactions)

•	Eventual consistency scales (but harder to reason about)

•	Use strongest consistency that works for your use case

4\. Start Simple, Evolve

•	Day 1: Monolithic

•	Day 100: Add async processing

•	Day 1,000: Consider microservices (if needed)

5\. Loose Coupling Enables Evolution

•	Monolithic with direct calls: Hard to change

•	Monolithic with events: Easy to add features

•	Microservices with events: Easy to scale features





