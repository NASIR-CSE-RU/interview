# System Design & Architecture Q&A for Senior Laravel Developer Interviews

This document contains practical system design and architecture questions tailored for senior Laravel backend developers, along with concise but deep answers.

---

## 1. High-Level System Design Fundamentals

### Q1. How would you design a scalable Laravel-based REST API for a high-traffic product?

**Answer:**

I’d start by separating concerns into layers:

1. **Client → API Gateway / Load Balancer**  
   - Use Nginx/HAProxy or a managed load balancer in front of multiple Laravel app instances.
2. **App Layer (Laravel)**  
   - Stateless API: no session in local storage; use stateless auth (JWT, Sanctum tokens) or Redis-backed sessions.
   - Follow a layered architecture: Controllers → Services → Repositories → Models.
3. **Data Layer**  
   - Primary relational DB (MySQL/PostgreSQL) with read replicas for heavy reads.  
   - Redis for caching, rate limiting, queues, and ephemeral data.
4. **Async & Background Jobs**  
   - Use Laravel queues (Redis/RabbitMQ/SQS) for non-critical or heavy tasks: email, notifications, reporting, third-party integration.
5. **Scaling Strategy**  
   - Horizontal scaling: multiple PHP-FPM workers behind load balancer.  
   - Auto-scaling policies based on CPU, queue length, or response times.
6. **Observability**  
   - Centralized logs (ELK, Loki), metrics (Prometheus), and APM (Sentry, Laravel Telescope, etc.).

Key principles: **statelessness, horizontal scalability, caching, and async processing**.

---

### Q2. How do you decide between a monolith, modular monolith, and microservices for a Laravel project?

**Answer:**

- **Monolith:**  
  - Good for small to medium projects and teams.  
  - Fast to develop and deploy, simpler local setup.  
  - Becomes hard to maintain if everything is mixed together.

- **Modular Monolith (often my default for Laravel):**  
  - Single codebase, single deployment, **clear bounded contexts** (modules) within `app/` or `src/`.  
  - Each module (e.g., `Billing`, `Orders`, `Users`) has its own controllers, services, models, routes.  
  - Good compromise: domain separation without network overhead of microservices.

- **Microservices:**  
  - Use when you have **very large scale**, team autonomy needs, or very different tech requirements per domain.  
  - Pros: independent deployments, scaling per service, technology flexibility.  
  - Cons: complexity in communication, distributed tracing, data consistency (Sagas, events), DevOps overhead.

For most Laravel products, I’d start with a **modular monolith** with clear boundaries and evolve to microservices only when truly needed.

---

### Q3. What are the key non-functional requirements you consider when designing a Laravel system?

**Answer:**

- **Scalability** – horizontal scaling, caching, read replicas, queue usage.
- **Availability** – health checks, zero-downtime deploys, redundancy.
- **Performance** – DB indexing, optimized queries, caching, HTTP/2, compression.
- **Security** – auth/authorization, input validation, CSRF/XSS/SQL injection protection, secret management.
- **Reliability & Fault Tolerance** – retries, circuit breakers around external calls, DLQs for jobs.
- **Observability** – logging, metrics, tracing, structured logs, error tracking.
- **Maintainability** – clean architecture, SOLID, modularization, coding standards.
- **Extensibility** – feature toggles, configuration-driven behavior, plug-in style modules.

---

## 2. Laravel Architecture & Layering

### Q4. How would you structure a large Laravel codebase beyond the default MVC?

**Answer:**

I’d introduce domain-based folders and explicit layers:

- `app/Domain/User/Models`, `app/Domain/User/Services`, `app/Domain/User/Policies`, etc.
- `app/Http/Controllers/Api/V1/...`
- `app/Services/` for cross-domain services (e.g., `PaymentGateway`, `NotificationService`).
- `app/Repositories/` or domain-specific repos for DB access, or use query objects.
- `app/Actions/` or `app/UseCases/` for single use-case classes (e.g., `CreateOrderAction`).
- `app/Jobs/`, `app/Events/`, `app/Listeners/` for async/event-driven behavior.

The goals are:
- Keep controllers thin.
- Keep Eloquent models light (avoid putting all logic in models).
- Encapsulate business logic into **services/actions**.
- Encourage testability (services, actions, repositories easy to unit test).

---

### Q5. What is the difference between a Service class and a Repository in Laravel architecture?

**Answer:**

- **Repository**  
  - Encapsulates data access logic (queries to DB, external search engines, etc.).  
  - Provides an interface like `UserRepository::findByEmail()` or `OrderRepository::getPendingOrders()`.

- **Service**  
  - Encapsulates business logic / workflows, often coordinating multiple repositories and external services.  
  - Example: `OrderService::createOrder()` orchestrates validations, calls `ProductRepository`, `InventoryService`, `PaymentGateway`, `OrderRepository`.

In short: **Repository ≈ data access**; **Service ≈ business process**. This separation keeps logic clean and testable.

---

### Q6. How do you ensure your controllers stay thin in a big Laravel API?

**Answer:**

- Use **Form Request** objects for validation and authorization.
- Move business logic to **Service** or **Action** classes.
- Move DB queries into **Repositories** or dedicated query classes.
- Use **API Resources** for response transformation/serialization.
- Use **Policies** and **Gates** for authorization instead of inline checks.

Controller methods should mostly do: read input → call a service/action → return a resource/response.

---

### Q7. How do you version your APIs in a large Laravel system?

**Answer:**

- Use **URL versioning**, e.g. `/api/v1/users` and `/api/v2/users`.
- Separate controllers by namespace: `App\Http\Controllers\Api\V1`, `Api\V2`.
- Optionally share logic via services/actions so versions reuse underlying code.
- Deprecate old versions with proper timeline, and use feature flags or configuration to control behavior.
- Document differences (OpenAPI/Swagger) and maintain clear versioned docs.

---

## 3. Database & Data Modeling

### Q8. How do you design database schema for a multi-tenant Laravel application?

**Answer:**

There are three common approaches:

1. **Single DB, shared schema (tenant_id column)**  
   - Simple to manage, cheaper.  
   - Add `tenant_id` to every tenant-owned table.  
   - Always scope queries by `tenant_id` (e.g. global scopes, middleware).  
   - Requires strong data isolation at query level and tests.

2. **Single DB, separate schema per tenant**  
   - Each tenant has its own tables, usually with a schema or table prefix.  
   - Better isolation but more complex migrations and tooling.

3. **Separate DB per tenant**  
   - Strongest isolation, best for compliance.  
   - More overhead: connection management, migrations, and operations.

In Laravel, single-DB-with-tenant_id is common; I would implement:

- Middleware to resolve current tenant from domain/header/token.
- Apply a global scope (e.g. `TenantScope`) or use a tenancy package.
- Ensure all queries and jobs are tenant-aware.

---

### Q9. How do you handle database read/write scaling in Laravel?

**Answer:**

- Use **read/write splitting** in Laravel’s DB configuration (master + replicas).  
- Configure `read` and `write` connections in `config/database.php`.  
- Laravel automatically routes writes to master and reads to replicas.
- For consistency-sensitive operations, force using the write connection:
  - `DB::connection('mysql::write')->select(...)` or within a transaction.
- Use caching for heavy read endpoints to reduce DB load.
- Monitor replication lag; avoid reading stale data for critical flows.

---

### Q10. How do you avoid N+1 query problems in Eloquent?

**Answer:**

- Use **eager loading** (`with`, `load`, `loadMissing`) for relationships used in loops:
  - `Post::with('comments')->get()`.
- Use `withCount` for aggregate counts.
- Be careful with nested eager loading; only load what’s needed.
- Use **debug tools** like Laravel Debugbar or Telescope in dev to watch query counts.
- For large lists, combine pagination with eager loading.

---

### Q11. How do you handle heavy aggregation/reporting in Laravel without overloading the DB?

**Answer:**

- Push heavy aggregations to:
  - DB with **optimized SQL + proper indexes** (grouping on indexed columns).
  - **Materialized tables** or summary tables updated periodically (cron/job).
- Use background jobs to pre-compute reports and cache results in Redis or a read-optimized table.
- Use async exports (queue + notifications) for large CSV/Excel instead of blocking HTTP.
- If needed, offload to external analytical DB or warehouse (e.g., ClickHouse/BigQuery).

---

## 4. Caching & Performance

### Q12. What caching strategies do you use in a Laravel system?

**Answer:**

- **Application-level cache** using `Cache` facade/contract:
  - `Cache::remember('key', ttl, fn () => ...)` for expensive queries.
- **Per-user or per-tenant cache keys** to avoid data leakage.
- **Route caching** (`php artisan route:cache`) and **config caching** (`config:cache`) for faster bootstrap.
- **Query caching layer** for common read-heavy queries.
- **Response caching** for highly static endpoints (e.g., config/data that rarely changes).
- **HTTP caching** with proper headers (ETag, Last-Modified, Cache-Control) at gateway/CDN level.
- Use Redis or Memcached as a centralized cache store in production.

---

### Q13. How do you identify and debug performance bottlenecks in a Laravel app?

**Answer:**

- Use **profiling tools**:
  - Laravel Telescope, Debugbar, or framework-integrated APM (New Relic, Datadog, Sentry Performance).
- Analyze:
  - Slow queries (missing indexes, N+1, large result sets).
  - Slow external API calls (network latency).
  - High CPU routes (heavy loops, complex transforms).
- Use `EXPLAIN` on queries to inspect index usage.
- Benchmark endpoints using tools like `ab`, `wrk`, or k6.
- Add application-level timers around critical blocks (micro-timing).
- Check queue processing time for async tasks.

---

## 5. Queues, Events & Asynchronous Design

### Q14. How do you design a queue-based architecture in Laravel for a high-volume system?

**Answer:**

- Identify tasks that don’t need to be synchronous:
  - Emails, SMS, webhooks, ledger updates, reporting, search indexing.
- Use Laravel’s queue system with a scalable backend (Redis, RabbitMQ, SQS).
- Separate queues by criticality:
  - `high` (time-sensitive), `default`, `low` (non-critical).
- Use **Horizon** (for Redis) or custom workers to manage concurrency & priorities.
- Implement retries with backoff, set max attempts, and use **failed_jobs** or DLQ for dead letters.
- Make jobs **idempotent** (e.g., use unique keys, business idempotency keys).
- Use events → listeners → jobs for decoupling.

---

### Q15. What is idempotency and why is it important in queue and API design?

**Answer:**

Idempotency means **performing the same operation multiple times results in the same outcome as doing it once**.

It’s important because:

- Network calls can be retried (client or gateway).
- Jobs can be retried after failure.
- Users may double-click buttons or resubmit forms.

Implementation examples in Laravel:

- Use a unique `idempotency_key` (header or body) and store processed keys in DB/Redis.
- Use database unique constraints and `insert ignore` or `update-or-create` patterns.
- Check if the action has already been performed before applying side effects.

---

### Q16. How would you design an event-driven architecture in Laravel?

**Answer:**

- Use **Domain Events** (e.g., `UserRegistered`, `OrderPaid`) dispatched from your services.
- Create **Listeners** or **Jobs** to react to those events:
  - Send emails, update analytics, sync to third-parties.
- Store events if you need **audit trails** or **event sourcing**.
- Use queues for async listeners to avoid blocking HTTP.
- For cross-service communication, publish events to a message broker (RabbitMQ, Kafka, etc.) from Laravel and consume them in other services.

---

## 6. API Design, Security & Auth

### Q17. How would you design authentication and authorization for a large Laravel API?

**Answer:**

- **Authentication:**
  - Use Laravel Sanctum or Passport/JWT for token-based APIs.
  - For first-party SPAs/mobile, Sanctum with session or token cookies.
  - For third-party integrations, issue **personal access tokens** or OAuth clients.

- **Authorization:**
  - Use **Policies** for model-level permissions (`view`, `update`, `delete`).
  - Use **Gates** for general permission checks.
  - Use **roles & permissions** (e.g., `roles` table + `permissions` table) and map them via policies.
  - Cache permissions per user/tenant in Redis for performance.

- Secure with:
  - Strong password hashing (bcrypt/argon2).
  - Rate limiting for login and sensitive routes.
  - 2FA for admin/critical accounts.

---

### Q18. How do you secure a public Laravel API from abuse (rate limiting, DDoS, etc.)?

**Answer:**

- Use **Laravel’s rate limiter** (`RateLimiter` facade, middleware like `throttle`):
  - Limit by IP, user ID, or API key.
- At infra level, use:
  - WAF / API gateway (Cloudflare, AWS WAF, etc.) for IP blocking, bot protection, geo-fencing.
  - Connection limits and timeouts on web server (Nginx).
- Implement **API keys** for partner integrations.
- Validate input rigorously, avoid heavy operations on unauthenticated endpoints.
- Cache heavy public data and serve it from CDN where possible.
- Monitor for unusual spikes and have some form of circuit breaker or fail-open cache.

---

## 7. File Storage, CDN & Media

### Q19. How do you design file uploads & media handling in Laravel?

**Answer:**

- Use Laravel’s **Storage** abstraction (`Storage` facade) to store files in:
  - Local disk in dev, S3/MinIO/GCS in production.
- Store only file metadata + path in DB, not binaries.
- Use **pre-signed URLs** for direct uploads from frontend to S3/MinIO to offload PHP.
- Serve files through:
  - CDN (CloudFront, Cloudflare) for performance.
  - Proper cache headers and versioning (e.g., include hash in file names).

For large media or user-generated content, avoid routing all traffic through the app; rely on object storage and CDN.

---

## 8. Observability, Logging & Monitoring

### Q20. How do you design your logging and monitoring strategy for a Laravel system?

**Answer:**

- Use **structured logging** (JSON) with context:
  - `Log::info('Order created', ['order_id' => $id, 'user_id' => $userId]);`
- Centralize logs in ELK/Graylog/Loki for search and dashboards.
- Use Laravel’s channels (stack, slack, daily) to route critical logs.
- Integrate **error tracking** (Sentry, Bugsnag, etc.) for exceptions and performance traces.
- Export **metrics** (requests per second, error rate, p95 latency, queue lengths) to Prometheus-like systems.
- Use health checks for readiness and liveness on app and queue workers.

---

## 9. Deployment, Configuration & Environments

### Q21. How would you design the deployment pipeline for a Laravel application?

**Answer:**

- Use CI/CD (GitHub Actions, GitLab CI, etc.):
  1. Run tests (PHPUnit, Pest) and static analysis (PHPStan, Psalm).
  2. Build artifacts (composer install with `--no-dev`, cache dependencies).
  3. Deploy to servers/containers (Docker/Kubernetes or managed PaaS).
  4. Run migrations (careful with backward compatibility).
  5. Clear & rebuild caches (`config:cache`, `route:cache`, `view:cache`).

- Use **blue-green or rolling deployments** to reduce downtime.
- Use **env-specific configurations** in `.env` and config files, never hard-code secrets.

---

### Q22. How do you manage configuration and secrets in Laravel at scale?

**Answer:**

- Use `.env` files for local dev only.
- In production, prefer:
  - Environment variables injected by the platform (Kubernetes secrets, Docker secrets, cloud secret managers).
- Never commit secrets to Git.
- Use `config()` files to centralize configuration, and cache them with `config:cache`.
- Rotate secrets periodically and design services to handle secret rotation gracefully.

---

## 10. Architecture Trade-offs & Patterns

### Q23. When would you move part of a Laravel monolith into a separate microservice?

**Answer:**

I’d consider it when:

- A module has **distinct scaling needs** (e.g., reporting engine, video processing).
- A team needs **independent deployment cadence** and ownership.
- There is a **different tech stack requirement** (e.g., real-time streaming best done in Node/Go).
- The module has become **a bottleneck** and its lifecycle differs from the core monolith.

Before doing that, I’d ensure:

- Clear **bounded context** and interfaces.
- Well-defined contracts (REST/gRPC/events).
- Observability and traceability between services.
- A plan for handling **distributed transactions** (Sagas, outbox pattern, eventual consistency).

---

### Q24. How do you apply the 12-Factor App principles in a Laravel project?

**Answer:**

- **Codebase:** Single codebase in Git; multiple environments.
- **Dependencies:** Managed via Composer; no global dependencies.
- **Config:** Stored in environment variables, not in code.
- **Backing services:** Treat DB, cache, queue, mail, storage as attached resources.
- **Build/Release/Run:** Separate steps; build artifact promoted across environments.
- **Processes:** Stateless app processes; no local session/state.
- **Port binding:** App exposed via web server / container.
- **Concurrency:** Scale out by running more processes/containers.
- **Disposability:** Fast startup/shutdown; use queues that handle reconnects.
- **Dev/Prod parity:** Keep environments as similar as possible.
- **Logs:** Treat logs as event streams.
- **Admin processes:** Run one-off admin tasks as separate processes (e.g., artisan commands).

---

### Q25. Describe a real system you’ve designed with Laravel and what key architectural decisions you made.

**Answer (example structure you can adapt in interviews):**

- **Context:** “I designed a Laravel-based multi-tenant SaaS for X domain with ~N daily active users.”
- **Architecture:** Modular monolith with domains (Billing, Users, Subscriptions, Reporting).
- **Data:** Single DB per environment with tenant_id and strict scoping.
- **Scaling:** Auto-scaled app servers behind load balancer, Redis cache, and queues for heavy tasks.
- **Key Decisions:**
  - Introduced **event-driven modules** for billing and notifications.
  - Implemented **idempotent ledger pipeline** (outbox + queues) to ensure financial correctness.
  - Used **API Resources** and versioning for public API.
  - Applied **Horizon** and rate limiting; added dashboards for observability.
- **Results:** Measurable improvements (reduced p95 latency, improved SLA, etc.).

In an interview, give a concrete story, including trade-offs, incidents, and what you learned.

---

You can extend this document with more domain-specific scenarios (fintech, e-commerce, booking, etc.) and practice answering each question aloud, focusing on **trade-offs**, not just “what” but **why** you chose each approach.