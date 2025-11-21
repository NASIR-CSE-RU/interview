# Microservices Architecture – Laravel Senior Dev Interview Q&A

> Focus: questions you might face as a **Senior Laravel Developer** working in a microservices environment.

---

## 1. Fundamentals

### Q1. What is microservice architecture, and how is it different from a Laravel monolith?

**Answer:**

- **Monolith:** One Laravel codebase, one database. All modules (auth, orders, payments, etc.) live in the same app and are deployed together.
- **Microservices:** Many **small, independent services**, each owning a specific business capability (Auth, User, Order, Payment, etc.), often with its **own database** and deployed independently.
- Key differences:
  - **Deployment:** Monolith → single deploy; Microservices → independent deploy per service.
  - **Scaling:** Monolith scales as a whole; Microservices can scale “hot” services only (e.g., Order).
  - **Tech stack:** Monolith → usually single stack; Microservices → different stacks per service (Laravel, Node, Go…).
  - **Data:** Monolith shares a single DB; Microservices usually enforce “**database per service**”.
- Trade-off: microservices give **better scalability and isolation**, but add **complexity** (distributed systems, observability, networking, consistency).

---

### Q2. When would you **not** recommend microservices for a Laravel project?

**Answer:**

- When the product is early-stage and requirements are changing fast.
- When the team is small and doesn’t have strong DevOps experience.
- When the system traffic and domain complexity are still manageable in a monolith.
- When observability, monitoring, CI/CD, and infra automation are weak.
- Rule of thumb: **Start with a well-modularized monolith** and only move to microservices when:
  - Teams are blocked by monolith coupling, or
  - Scaling/complexity issues are real, not hypothetical.

---

## 2. Service Boundaries & Design

### Q3. How do you decide **service boundaries** when splitting a Laravel monolith into microservices?

**Answer:**

- Use **Domain-Driven Design (DDD)** ideas:
  - Identify **bounded contexts** (e.g., Identity, Billing, Orders, Inventory).
- Group things that change together into the same service:
  - If every billing change affects “invoice + payment” together, keep them in the same service.
- Avoid splitting purely by technical layers (e.g., “Controllers service”, “Repositories service”).
- Ask:
  - Does this service have a **clear single responsibility**?
  - Does it have its **own data** that others shouldn’t modify directly?
  - Is its **API understandable** from outside (e.g., `/orders`, `/payments`)?

---

### Q4. In a microservices system, can multiple Laravel services share the **same database**?

**Answer:**

- Recommended pattern: **No** – each service should own its **own schema / DB**.
- Why:
  - To avoid **tight coupling** via shared tables.
  - To enforce clear **ownership** and avoid cross-service joins.
  - So you can evolve each service independently (migrations, refactors).
- If you must share (legacy):
  - Start by **logically separating** schemas (different DBs) even on the same server.
  - Don’t let services write into each other’s tables.
  - Plan a **gradual migration** to fully separate DBs.

---

### Q5. How do you handle **queries that need data from multiple services** (e.g., Order + User + Payment info)?

**Answer:**

Common patterns:

- **API Composition / Aggregator service:**
  - A “BFF” / Gateway or dedicated Aggregator calls multiple services (Order, User, Payment) and merges responses.
- **CQRS and read models:**
  - Each service emits events (e.g., `OrderPlaced`, `UserUpdated`, `PaymentCaptured`).
  - A separate **Reporting/Read Model service** subscribes and builds denormalized tables for dashboards.
- **Don’t:** Do cross-service DB joins or let one service query another service’s database directly.

In Laravel:

- Use **HTTP client** (`Http::get(...)`) or **queues/events** to collect data.
- Use dedicated **read models** in a Reporting Laravel app for complex reports.

---

## 3. Communication Between Services

### Q6. When would you use **synchronous HTTP** vs **asynchronous messaging** between microservices?

**Answer:**

- **Synchronous HTTP (REST/gRPC)**:
  - When you need an immediate response (e.g., “check user balance before placing order”).
  - When the operation is **query-like** (read data).
  - Simple to implement: `Http::get/post` in Laravel.
- **Asynchronous messaging (queues/events)**:
  - For background tasks and workflows (send email, create invoice, record ledger).
  - For **event-driven** communication (`OrderPlaced` → Payment/Notification services react).
  - To decouple services and improve resilience (producer doesn’t wait).
- Rule:
  - Use **HTTP for queries** (read), **messages for side-effects and workflows** (write/work).

---

### Q7. How would you implement **service-to-service HTTP calls** in Laravel safely?

**Answer:**

- Use Laravel’s **HTTP client**:

  ```php
  $response = Http::timeout(2)
      ->retry(3, 100)
      ->withToken($serviceToken)
      ->get(config('services.payment.base_url').'/health');
  ```

- Best practices:
  - Set **timeouts** (never infinite waits).
  - Use **retry with backoff** for transient errors.
  - Pass **correlation/request IDs** for tracing.
  - Secure with **service tokens** or mTLS, not just open internal URLs.
  - Handle errors gracefully (circuit breaker/fallback).

---

### Q8. How do you implement **asynchronous communication** in Laravel microservices?

**Answer:**

- Use **queues** (Redis, RabbitMQ, SQS, etc.) with Laravel’s **Job** system.
- Flow:
  - Service A dispatches a job or publishes an event (e.g., `OrderPlaced`).
  - Service B subscribes (consumer) and handles the job/event.
- Tools:
  - `dispatch(new ProcessOrder($orderId));`
  - **Horizon** for queue management / monitoring.
- For cross-service messaging:
  - Use a **central message broker** (RabbitMQ/Kafka).
  - Use **topic/route keys** for routing events.
  - Serialize payloads as JSON or Avro, with clear contracts.

---

## 4. Auth, Security & API Gateway

### Q9. How do you handle **authentication** across multiple Laravel microservices?

**Answer:**

- Centralize **identity** in an **Auth/Identity service**:
  - Users log in via Auth service and obtain a **JWT / opaque token**.
- Other services:
  - Validate tokens via shared secret/public key (for JWT) or introspection endpoint.
  - Don’t store session in every service; treat token as **self-contained** identity.
- In Laravel:
  - Use **Laravel Passport**, **Sanctum**, or custom JWT guards.
  - Implement a `UserContext` resolved from token claims for each request.

---

### Q10. What is an **API gateway**, and how would you use it with Laravel microservices?

**Answer:**

- API Gateway:
  - A single entry point that routes requests to underlying services.
  - Responsibilities:
    - Routing (`/api/orders` → Order service).
    - Auth & rate limiting.
    - Request/response transformation.
    - CORS, logging, metrics.
- Options:
  - **External gateways:** Kong, Nginx, Traefik, AWS API Gateway.
  - **Custom Laravel gateway:** A small Laravel app that proxies requests to internal services.
- For Laravel:
  - Frontend → Gateway (Laravel / Kong) → internal Laravel services.
  - Gateway verifies token once, then forwards user context to downstream services.

---

## 5. Data Consistency & Transactions

### Q11. Why can’t you use a single **database transaction** across multiple microservices? How do you handle it?

**Answer:**

- A DB transaction is **local** to one database/connection.
- Microservices typically each have their own DB; you cannot have a truly ACID transaction across them without **2PC** (two-phase commit), which is complex and discouraged.
- Solution: **Eventual consistency** using patterns like:
  - **Saga pattern:** Break business process into local transactions with compensating actions.
  - **Outbox pattern:** Persist events in the same DB transaction, then publish them via a reliable background process.
- Example (Laravel):
  - Order service:
    - `DB::transaction()` → create order + insert `outbox` record.
  - A Laravel job reads `outbox` table and publishes `OrderPlaced` to message broker.
  - Other services react and update their own state.

---

### Q12. Explain the **Saga pattern** with a Laravel example.

**Answer:**

- Saga: A **sequence of local transactions** across services, coordinated via events or a central orchestrator, with **compensating actions** if something fails.
- Example: “Place order with payment”

  1. Order service creates order with `status = PENDING`, publishes `OrderCreated`.
  2. Payment service charges card, publishes `PaymentSucceeded` or `PaymentFailed`.
  3. Order service listens:
     - On `PaymentSucceeded` → set status `CONFIRMED`.
     - On `PaymentFailed` → set status `CANCELLED` and maybe trigger refund/rollback actions.

- In Laravel:
  - Use jobs + events + listeners across services.
  - Implement compensating logic in listeners (e.g., “cancel order”, “release stock”).

---

### Q13. What is the **Outbox pattern**, and how would you implement it in Laravel?

**Answer:**

- Problem: If you do `DB::commit()` and then publish to a queue, one could succeed and the other fail → **inconsistency**.
- Outbox pattern:
  - Within a single DB transaction, you:
    1. Persist your business entity (e.g., `orders`).
    2. Persist an **outbox record** (e.g., `outbox` table) representing the event to be sent.
  - A separate background process reads outbox records and publishes them to the message broker, then marks them as **sent**.
- Laravel implementation:
  - In service layer:

    ```php
    DB::transaction(function () use ($data) {
        $order = Order::create($data);
        Outbox::create([
            'type' => 'order.created',
            'payload' => json_encode($order->toArray()),
        ]);
    });
    ```

  - A scheduled job or queue worker:
    - Reads unsent outbox rows, publishes them to RabbitMQ/Kafka, updates status.

---

### Q14. What is **idempotency**, and why is it important in microservices?

**Answer:**

- **Idempotent** operation: calling it multiple times has the **same effect** as calling it once.
- Why important:
  - Network retries can cause duplicate requests/messages.
  - Consumers might process the same message more than once.
- In microservices:
  - Avoid **double charging** payment, **double creating** orders, etc.
- Laravel examples:
  - Use **idempotency keys** (header or request ID) stored in DB with unique constraint.
  - Use **unique indexes** (e.g., `unique (user_id, external_id)`) to enforce one-time operations.
  - On queues, store processed message IDs in a table/cache before executing.

---

## 6. Reliability, Resilience & Observability

### Q15. How do you handle **failures** between services (e.g., Payment service is down)?

**Answer:**

- Techniques:
  - **Timeouts**: never wait too long for HTTP calls.
  - **Retries with backoff**: retry transient failures (e.g., 5xx) with delays.
  - **Circuit breaker**:
    - If a service keeps failing, open circuit and fail fast for some time instead of hammering it.
  - **Fallbacks**:
    - Limited degraded behavior (e.g., show cached data / disable feature).
- In Laravel:
  - Configure `Http::retry()` with reasonable backoff.
  - Use libraries/patterns for circuit breaking (via middleware or specialized clients).
  - Log failures with correlation IDs for debugging.

---

### Q16. How do you handle **logging, tracing and metrics** in a Laravel microservices ecosystem?

**Answer:**

- **Logging**:
  - Use a **centralized logging system** (ELK, Loki, etc.).
  - Standardize log format (JSON) and always include:
    - `correlation_id`, `service_name`, `request_id`, `user_id`.
- **Tracing**:
  - Use **distributed tracing** (OpenTelemetry, Jaeger, Zipkin).
  - Generate a trace ID at the gateway and pass it via headers to all services; each service logs this ID.
- **Metrics**:
  - Collect service metrics (latency, error rate, queue depth) via Prometheus/StatsD, etc.
- Laravel:
  - Use `logging.php` channels for centralized logging.
  - Middleware to add `X-Request-Id` or `X-Correlation-Id` header to every incoming request.
  - Use packages for OpenTelemetry, or custom middleware to push spans/metrics.

---

## 7. Deployment, Versioning & Testing

### Q17. How do you deploy Laravel microservices without breaking other services?

**Answer:**

- Use **backward-compatible changes**:
  - Don’t remove fields immediately; deprecate first.
  - Don’t change API contracts in a breaking way without versioning.
- Deployment strategies:
  - **Blue–green** or **rolling** deployments (Kubernetes, etc.).
  - Schema migrations:
    - **Phase 1:** Add new columns / tables without dropping old ones.
    - **Phase 2:** Deploy code to use new schema.
    - **Phase 3:** Cleanup old schema only when all services are migrated.
- Automation:
  - CI/CD pipeline per service.
  - Automated tests + smoke tests after deploy.

---

### Q18. How do you version APIs in a microservices system built with Laravel?

**Answer:**

- **URL versioning**:
  - `/api/v1/orders`, `/api/v2/orders` routes in Laravel.
- **Header-based versioning**:
  - `Accept: application/vnd.myapp.v2+json`.
- Strategy:
  - Keep v1 stable while introducing v2.
  - Gradually migrate clients to v2 and then deprecate v1 once usage drops.
- Laravel:
  - Use **route groups** per version: `Route::prefix('v1')->group(...)`.
  - Maintain separate controllers or transformers per version.

---

### Q19. What testing strategy do you follow for Laravel microservices?

**Answer:**

- **Unit tests**:
  - Test domain logic (services, value objects, policies).
- **Integration tests**:
  - Test DB, queues, external APIs within a single service.
- **Contract / Consumer-driven tests**:
  - Verify that providers (services) honor the API contract expected by consumers.
  - E.g., Pact-like tests.
- **End-to-end tests**:
  - Test flows across multiple services via gateway.
- Practical Laravel tools:
  - PHPUnit/Pest for unit & feature tests.
  - Use in-memory or test DBs, `RefreshDatabase` trait.
  - Use docker-compose for local E2E environment with multiple Laravel services + DB + broker.

---

## 8. Local Development & Migration Strategy

### Q20. How do you manage **local development** for multiple Laravel microservices?

**Answer:**

- Use **docker-compose**:
  - Each service as a container, plus shared infra (MySQL, Redis, RabbitMQ, etc.).
- Local domains:
  - `auth.local.test`, `orders.local.test`, etc., mapped via `/etc/hosts` or Traefik/Nginx.
- Shared `.env.example` templates to standardize config.
- Seed **test data** for each service to make flows testable.
- Optionally use a small local **API gateway** (Laravel or Traefik) to route requests.

---

### Q21. How would you **gradually migrate** a Laravel monolith to microservices (Strangler Fig pattern)?

**Answer:**

High-level steps:

1. **Stabilize the monolith**:
   - Improve modularization (modules, domain layers, service classes).
   - Add tests & monitoring.
2. Identify a **clear bounded context** that can be extracted (e.g., Payments, Notifications).
3. Create a new **Laravel microservice** for that context:
   - Move logic + create a new DB schema (or share temporarily in transition).
   - Build an API that the monolith can call.
4. **Strangle** the monolith:
   - Replace internal monolith calls with HTTP/queue calls to the new service.
   - Gradually move more endpoints to microservices.
5. Repeat for other domains until the monolith becomes simpler or disappears.

---

### Q22. What are the most common **pitfalls** you’ve seen (or expect) when using microservices with Laravel?

**Answer:**

- Splitting into microservices **too early** without real need.
- Treating microservices like “**distributed monolith**”:
  - Shared DB, shared models, tight coupling.
- Poor **observability**:
  - No centralized logs, no tracing → debugging is painful.
- Ignoring **idempotency and retries**:
  - Leads to double processing and inconsistent data.
- Over-chatty services:
  - Too many small synchronous calls causing latency and cascading failures.
- Underestimating **DevOps complexity**:
  - Without good CI/CD, monitoring, and infra automation, microservices hurt more than help.

---

This markdown can be saved directly as `microservices-laravel-senior-qa.md` and used as a study sheet for interviews.
