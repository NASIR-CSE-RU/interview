# Senior Laravel Technical Discussion – Questions & Answers

This document contains *discussion-style* questions and answers you can expect in a **Senior Laravel Developer** interview.  
They are meant to sound like realistic conversation in a technical round, not just theory.

---

## 1. Overall Architecture & Code Organization

### Q1. If we look into your last big Laravel project, how did you structure the codebase beyond default MVC?

**Answer:**

I avoid putting all logic into controllers and models. I prefer a **domain-based modular structure**:

- `app/Domain/User`, `app/Domain/Order`, `app/Domain/Billing` etc.
- Each domain contains `Models`, `Services`, `Policies`, sometimes `Actions`, `Events`, `Listeners`.
- `app/Http/Controllers/Api/V1/...` for transport layer only.
- `app/Actions` or `app/UseCases` for one-class-per-use-case (e.g. `CreateOrderAction`).
- `app/Repositories` or domain-specific repositories for DB access when queries are complex.

This keeps controllers thin, business logic centralized in services/actions, and makes testing and onboarding much easier.

---

### Q2. How do you decide what goes into a Service class vs a Model vs a Controller?

**Answer:**

- **Controller**: HTTP orchestration only → validate input, call a service/action, return response.
- **Model (Eloquent)**: Entity representation + simple relations & attribute casting. I keep business rules light here.
- **Service/Action**: Orchestrates business logic, coordinates multiple models/repositories, handles workflow steps.

If I see a controller method larger than ~20–30 lines or doing heavy business logic, it should go into a service/action. If a model contains complex multi-step logic that depends on other models or external systems, I move it into a service.

---

### Q3. How would you refactor a legacy Laravel codebase where controllers and models are “fat” and everything is tightly coupled?

**Answer:**

Steps I usually follow:

1. **Identify hotspots** (most frequently changed/buggy controller methods).
2. Extract core business logic into **Service** or **Action** classes.
3. Move validation into **Form Request** classes.
4. Introduce **Policies** for authorization logic scattered around.
5. Gradually move complex queries to **Repositories** or query objects.
6. Add **unit tests** around the new services/actions to protect behavior.
7. Introduce a **domain-based folder structure** so future code lands in the right place.

I don’t try to “big bang” refactor everything; I refactor and improve as we touch those areas for new features or bug fixes.

---

## 2. Eloquent, Relationships & DB Layer

### Q4. How do you approach designing Eloquent models and relationships in a complex domain?

**Answer:**

- Start from the **domain model** and business language: entities (Order, Invoice, Payment), value objects, aggregates.
- Map core entities to Eloquent models, with clear relationships:
  - `hasOne`, `hasMany`, `belongsTo`, `belongsToMany`, `morphOne`, `morphMany`.
- Keep model responsibilities:
  - Table mapping, relationships, attribute casts, scopes.
- Use **query scopes** for common filters (`active()`, `published()`, `forTenant($id)`).
- For very complex queries, I prefer `Repository` or dedicated query classes instead of bloating the model.

---

### Q5. How do you avoid N+1 issues and memory bloat when using Eloquent?

**Answer:**

- Use **eager loading**: `with()` / `load()` / `loadMissing()` for relations used in loops.
- Use **`withCount()`** for counts instead of loading entire relationships.
- For large datasets:
  - Use `chunk()`, `cursor()`, or `lazy()` to process records in smaller batches.
  - For reporting, rely on SQL aggregates rather than loading big collections into PHP.
- In development, use **Telescope**, **Debugbar**, or query logging to detect N+1 queries early.

---

### Q6. Have you ever tuned a slow query in Laravel? Walk me through your process.

**Answer:**

Typical process:

1. Reproduce the slow endpoint and capture the **exact SQL** query (Telescope, Debugbar, DB::listen).
2. Run `EXPLAIN` in MySQL/Postgres to see if indexes are used or if it’s scanning the whole table.
3. Add or adjust **indexes** on filtering/join columns (e.g., `status`, `tenant_id`, `created_at`).
4. Avoid functions on columns in `WHERE` when possible.
5. Replace unnecessary `SELECT *` with selected fields; avoid loading heavy relationships if not needed.
6. For super heavy reports, move to:
   - Pre-aggregated tables,
   - Materialized views,
   - Or scheduled jobs that compute data and cache it.

After changes, re-measure response time and check if we reduced CPU/IO usage.

---

## 3. Caching & Performance in Real Systems

### Q7. How do you use caching in Laravel for real-world performance gains?

**Answer:**

- Use `Cache::remember()` for expensive queries (e.g., config tables, dashboard aggregates).
- Cache by **tenant, user, role, or locale** in the cache key to avoid leaking data.
- Use Redis as a centralized store in production.
- For complex pages, sometimes cache the **entire response** (response caching) and invalidate on change.
- Use **route cache**, **config cache**, and **optimized autoloader** in production deployments.
- Combine application-level cache with **HTTP caching headers** and a CDN for static content.

---

### Q8. Imagine one API endpoint becomes slow in production. How would you debug it step by step?

**Answer:**

1. Check monitoring/APM (Datadog, Sentry, Telescope, logs) to confirm:
   - Response time,
   - Error rate,
   - Throughput.
2. Inspect the specific route:
   - Count DB queries,
   - Look for N+1 or heavy joins.
3. Run `EXPLAIN` on slow queries and fix indexing.
4. Check external dependencies (payment gateway, third-party API) – they often cause spikes.
5. Add simple timing logs around major sections of code to pinpoint bottlenecks.
6. Introduce caching or async processing where appropriate.
7. Validate PHP-FPM and DB resource limits – sometimes we need more workers or better pooling.

---

## 4. Queues, Jobs & Async Processing

### Q9. What real-world problems have you solved using queues in Laravel?

**Answer:**

I’ve used queues for:

- Email/SMS/Push notifications after user actions.
- Asynchronous **ledger updates** or financial reconciliations.
- Syncing data to third-party services (CRM, analytics).
- Generating PDF reports and sending download links.
- Processing large imports/exports.

Queues reduced request latency, improved user experience, and allowed us to scale workers independently from web traffic.

---

### Q10. How do you design a robust job in Laravel so it behaves well under retries and failures?

**Answer:**

- Make job **idempotent** (e.g., check if an action is already applied before applying again).
- Use a **stable business key** (order id, reference) to ensure only one job updates a resource at a time.
- Catch and handle expected exceptions (e.g., transient network issues) and rethrow unexpected ones.
- Configure:
  - Max attempts,
  - Backoff strategy,
  - Separate queues for critical vs low-priority jobs.
- Log enough context (job id, entity id, tenant) to debug failed jobs.
- For high importance actions, consider a **dead-letter queue** or manual review of failed jobs.

---

### Q11. Have you used Laravel Horizon? How do you use it in a production environment?

**Answer:**

Yes, mainly when using Redis as a queue:

- Define **queue workers and supervisors** with different queues and priorities.
- Monitor:
  - Throughput,
  - Failed jobs,
  - Processing time.
- Use **Horizon’s dashboard** to adjust concurrency and balance workloads.
- Protect Horizon with authentication (e.g., only staff access).
- Configure alerts for high failure rates or stuck queues.

Horizon gives me both control and visibility over background jobs.

---

## 5. Authentication, Authorization & Security

### Q12. How do you typically implement authentication and authorization in a large Laravel app?

**Answer:**

- Authentication:
  - For API: **Sanctum** (tokens) or Passport/JWT, depending on integration needs.
  - For web: session-based auth with standard guards.
- Authorization:
  - Use **Policies** for model-level permissions (`view`, `update`, `delete`).
  - Use **Gates** for business-wide checks.
  - Implement roles & permissions (e.g., `roles` & `permissions` tables) and map them via policies.
- Security:
  - Input validation via **Form Requests**.
  - CSRF for web, rate limiting for auth endpoints.
  - Password hashing (bcrypt/argon2), 2FA for admin roles.
  - Regularly review third-party packages and keep framework up-to-date.

---

### Q13. How do you handle multi-role or multi-tenant permissions in Laravel?

**Answer:**

- Maintain **User–Role–Permission** model, optionally scoped by tenant.
- Build a small ACL system:
  - `roles` table,
  - `permissions` table,
  - pivot `role_permission` & `user_role`.
- Cache permissions per user/tenant in Redis.
- Use **Policies** to check permissions in a consistent way:
  - e.g., `OrderPolicy@update` checks both role and tenant ownership.
- For multi-tenant:
  - Always scope queries by `tenant_id`.
  - Ensure that authorization checks also validate tenant boundaries.

---

### Q14. Can you describe a security incident or risk you mitigated in a Laravel app?

**Answer (example pattern you can adapt):**

- We had an endpoint returning too much user data (PII) due to broad serialization.
- I fixed it by:
  - Introducing **API Resources** to control exact fields exposed.
  - Adding **Policies** for access control.
  - Removing sensitive columns from `visible` attributes and adding encryption for some fields.
- Then we added security review steps to PRs and auditing of logs for sensitive endpoints.

---

## 6. REST API Design & Integration

### Q15. How do you design APIs in Laravel that are easy to maintain and evolve?

**Answer:**

- Versioning in URL (`/api/v1/...`).
- Use **Form Requests** for validation and some authorization.
- Use **API Resources** for response normalization and versioning.
- Follow REST conventions:
  - `GET /orders`, `POST /orders`, `PATCH /orders/{id}`, `DELETE /orders/{id}`.
- Keep controllers thin; logic goes into services/actions.
- Provide good error structures (standard JSON error format).
- Document with **OpenAPI/Swagger** and automatically generate docs if possible.

---

### Q16. How do you handle breaking changes in your API?

**Answer:**

- Introduce new **version** (e.g., `/v2`) with updated behavior.
- Keep `/v1` working for existing clients for a deprecation period.
- Communicate timeline for deprecation; provide migration guide.
- Internally, reuse services/actions where possible; controllers/resources differ.
- Use **feature flags** to gradually roll out new behavior.

---

## 7. Testing, Quality & CI/CD

### Q17. What is your approach to testing in Laravel – which layers do you focus on?

**Answer:**

- **Unit tests** for services, value objects, and pure business logic (fast and reliable).
- **Feature tests** for HTTP endpoints: checking status, structure, and side effects.
- Some **integration tests** that touch DB/queue when necessary.
- Use factories and seeders for consistent test data.
- In CI, run tests on every PR; don’t merge to main unless tests pass.

I like to have at least critical paths (auth, payment, orders, etc.) covered by tests.

---

### Q18. How do you integrate Laravel projects with CI/CD?

**Answer:**

- Use GitHub Actions, GitLab CI, or similar:
  1. On PR: run PHP linting, static analysis (PHPStan/Psalm), and tests.
  2. On merge to main: build artifact (composer install `--no-dev`), run migrations in a safe way.
  3. Deploy via SSH, Docker, or Kubernetes.
- Post-deploy:
  - Run `php artisan config:cache`, `route:cache`, `view:cache`.
  - Ensure horizon/queue workers restart if needed.
- Use health checks and gradually shift traffic (blue-green, rolling deploys).

---

## 8. Logging, Monitoring & Troubleshooting

### Q19. How do you implement logging and error tracking in Laravel?

**Answer:**

- Use Laravel’s logging channels:
  - `stack` channel for normal logs,
  - daily log rotation,
  - Slack/Telegram channel for critical alerts.
- Use structured logs (context arrays).
- Integrate **Sentry/Bugsnag** for:
  - Error aggregation,
  - Performance monitoring,
  - Alerts.
- For debugging production issues:
  - Correlate logs using request IDs or user IDs.
  - Use Telescope or custom debugging only in non-production or behind protection.

---

### Q20. Can you walk me through how you debug a tricky production bug?

**Answer:**

1. Reproduce locally if possible with same input; otherwise, collect logs and context.
2. Check error tracking (Sentry) to see stack traces, user context, frequency.
3. Add more **contextual logs** around suspicious areas.
4. If needed, add a feature flag and temporarily disable risky feature.
5. After identifying the root cause:
   - Write a test that reproduces the bug,
   - Fix it,
   - Validate in staging,
   - Deploy with monitoring.
6. Post-mortem: document what happened, add safeguards (validation, type checks, better defaults).

---

## 9. Multi-Tenancy, Scaling & Architecture Decisions

### Q21. Have you built or maintained a multi-tenant Laravel app? How did you handle isolation?

**Answer:**

Yes. Main patterns I’ve used:

- **Single DB + tenant_id** column on tenant-specific tables.
- Middleware to resolve current tenant from domain or header.
- Global scopes or tenant-aware repositories to always filter by `tenant_id`.
- Separate queues per tenant where necessary or encode tenant id in job payload.
- Tenant-scoped caching keys.
- Separate object storage folders/buckets per tenant for files.

We also have admin tools to inspect and operate per-tenant data when needed.

---

### Q22. When do you decide to split a Laravel monolith into microservices?

**Answer:**

Only when:

- A domain has very different **scaling characteristics** (e.g., reporting engine).
- We need a different tech stack (real-time, heavy streaming).
- Teams are blocked because deployments are tightly coupled.

Before splitting:

- Identify clear **bounded contexts**.
- Define APIs/contracts/events between services.
- Implement tracing and shared logging so debugging across services is feasible.
- Plan for data consistency (Sagas, outbox pattern).

---

## 10. Leadership, Code Review & Collaboration

### Q23. As a senior Laravel engineer, how do you handle code reviews?

**Answer:**

- Focus on:
  - Correctness,
  - Readability,
  - Performance,
  - Security,
  - Domain clarity.
- Check:
  - Separation of concerns,
  - Proper use of Laravel features,
  - Tests for critical paths.
- Give **constructive feedback**, explain *why* something is a concern.
- Approve quickly when changes are small and isolated; ask for iterative improvement for larger refactors.

---

### Q24. How do you mentor junior developers in Laravel best practices?

**Answer:**

- Pair programming on tricky tasks.
- Short internal sessions on:
  - Eloquent tricks,
  - Query optimization,
  - Form Requests,
  - Policies,
  - Testing.
- Review their PRs with examples and links to docs.
- Provide simple guidelines:
  - Keep controllers thin.
  - Avoid heavy logic in views.
  - Write tests for bugs they fix.
  - Prefer explicit over “clever” code.

---

### Q25. What are some mistakes you made earlier with Laravel that you don’t do anymore?

**Answer:**

- Putting too much logic into controllers and models.
- Not monitoring N+1 issues early.
- Skipping tests for “small” features that later broke something else.
- Using facades everywhere instead of injecting dependencies for testability.
- Ignoring database indexing until performance became a crisis.

Now I try to think more about **domain modeling, performance from day one**, and keeping the codebase maintainable as the team grows.

---

You can practice by **answering these questions in your own words**, using examples from your real projects. In a senior-level interview, the *how* and *why* behind your decisions matter more than just Laravel syntax.
