# Laravel Service Container & Dependency Injection – Senior Interview Q&A

> You can save this as `laravel-service-container-di.md` for your prep.

---

## 1. Fundamentals

### Q1. What is the Laravel service container and why does it exist?

**Answer:**

The **service container** is Laravel’s central place for **managing class dependencies** and performing **dependency injection**.

- It’s basically an IoC (Inversion of Control) container.
- Instead of you manually doing `new SomeClass(...)` everywhere, you **declare dependencies** (via type-hints), and the container:
  - Figures out how to construct them (via reflection).
  - Reuses shared instances if needed (singletons/scoped).
- It decouples **“how to build an object”** from **“where the object is used”**, which:
  - Makes code more testable (swap implementations/mocks).
  - Makes large apps easier to maintain and refactor.

---

### Q2. What is dependency injection, and how does Laravel support it?

**Answer:**

**Dependency Injection (DI)** means you **receive dependencies from outside**, instead of creating them inside the class.

In Laravel:

```php
class OrderController extends Controller
{
    public function __construct(
        private PaymentService $paymentService,
    ) {}

    public function store(Request $request)
    {
        $this->paymentService->charge(...);
    }
}
```

- You only type-hint `PaymentService`.
- Laravel’s container **resolves** `PaymentService` and injects it.
- If `PaymentService` needs other dependencies, the container recursively resolves them too.

This gives:

- **Loose coupling** (swap `PaymentService` implementation easily).
- **Testability** (bind a fake in tests).
- **Cleaner code** (no `new` chains all over controllers).

---

### Q3. How is the service container different from a service provider or a “service class”?

**Answer:**

- **Service container**
  - Core object that resolves and manages dependencies.
  - Exposed via helpers like `app()`, `resolve()`, and auto-resolution.

- **Service provider**
  - A class where you **configure and register bindings** into the container.
  - Usually extends `Illuminate\Support\ServiceProvider`.
  - `register()` → define bindings; `boot()` → perform actions after all bindings are registered.

- **Service class**
  - Your own business-logic class (e.g., `FlightSearchService`).
  - It is **stored/managed by the container**, but not the container itself.

---

## 2. Binding & Resolution

### Q4. How does Laravel automatically resolve dependencies without explicit bindings?

**Answer:**

If you type-hint a **concrete class** that:

- Is instantiable (not abstract/interface).
- Has a resolvable constructor (its own dependencies can also be resolved),

then Laravel uses **reflection**:

1. Inspects constructor parameters.
2. For each class type-hint, recursively resolves that dependency.
3. For scalars, uses default values or throws if it can’t resolve.

Example (no explicit binding needed):

```php
class InvoiceService
{
    public function __construct(
        private LoggerInterface $logger, // needs binding
        private CacheManager $cache,     // auto-resolved
    ) {}
}
```

- `CacheManager` is a concrete class that Laravel knows how to build.
- `LoggerInterface` is an interface → you **must** bind it.

---

### Q5. When do you need to create explicit bindings in the container?

**Answer:**

You need explicit bindings when:

1. **You type-hint an interface or abstract class**:

```php
$this->app->bind(
    PaymentGateway::class,
    StripePaymentGateway::class,
);
```

2. **You need custom construction logic**:

```php
$this->app->bind(ExternalApiClient::class, function ($app) {
    return new ExternalApiClient(
        config('services.external_api.key'),
        $app->make(LoggerInterface::class),
    );
});
```

3. **You want to share the same instance** across the app (singleton/scoped).

4. **You want different implementations** in different contexts (contextual binding).

---

### Q6. Explain `bind`, `singleton`, `scoped`, and `instance` in the container.

**Answer:**

All of them are ways to tell the container **how to build and reuse** objects:

```php
// 1. bind: new instance every time
$this->app->bind(Foo::class, function ($app) {
    return new Foo();
});

// 2. singleton: one instance for the entire app lifecycle
$this->app->singleton(Bar::class, function ($app) {
    return new Bar();
});

// 3. scoped: one instance per request/job lifecycle (reset between requests)
$this->app->scoped(Baz::class, function ($app) {
    return new Baz();
});

// 4. instance: you give the container an existing instance
$service = new SomeService();
$this->app->instance(SomeService::class, $service);
```

- `bind` → transient, **new object every resolution**.
- `singleton` → **global shared** instance.
- `scoped` → shared within **one request / queue job**, then flushed.
- `instance` → use an already-created object (often for testing or bootstrapping).

For **request-specific state** (like “current tenant config”), `scoped` is usually safer than `singleton` in Octane/queue workers.

---

### Q7. How do you bind an interface to a concrete implementation and inject it?

**Answer:**

**Service provider:**

```php
use App\Contracts\SmsSender;
use App\Services\TwilioSmsSender;

public function register(): void
{
    $this->app->bind(SmsSender::class, TwilioSmsSender::class);
}
```

**Usage (constructor DI):**

```php
use App\Contracts\SmsSender;

class TwoFactorService
{
    public function __construct(private SmsSender $smsSender) {}

    public function sendCode(string $phone, string $code): void
    {
        $this->smsSender->send($phone, "Your code is {$code}");
    }
}
```

Now:

- Anywhere you inject `SmsSender`, Laravel provides `TwilioSmsSender`.
- In tests, you can replace binding with a fake implementation.

---

### Q8. What is contextual binding and why is it useful?

**Answer:**

**Contextual binding** allows you to provide **different implementations** of the same interface depending on **where** it’s being injected.

Example: `PaymentGateway` in different services:

```php
use App\Contracts\PaymentGateway;
use App\Services\StripeGateway;
use App\Services\PaypalGateway;

$this->app->when(\App\Services\CheckoutService::class)
    ->needs(PaymentGateway::class)
    ->give(StripeGateway::class);

$this->app->when(\App\Services\SubscriptionService::class)
    ->needs(PaymentGateway::class)
    ->give(PaypalGateway::class);
```

Why useful:

- Multi-tenant / region-based behaviors.
- Different implementations for web vs. mobile flows.
- A/B testing different gateways without changing constructors.

---

### Q9. What are container tags and when would you use them?

**Answer:**

**Tags** let you group multiple bindings under one label:

```php
$this->app->bind(ReportGenerator::class, SalesReportGenerator::class);
$this->app->bind(ReportGenerator::class, UserReportGenerator::class);

$this->app->tag(
    [SalesReportGenerator::class, UserReportGenerator::class],
    'report-generators',
);
```

Later:

```php
$generators = $this->app->tagged('report-generators');

foreach ($generators as $generator) {
    $generator->generate();
}
```

Use tags when:

- You want to run **all registered handlers** (e.g., post-processors, report generators, notification channels).
- You want to implement **plugin/extension** style architecture.

---

## 3. Service Providers & Lifecycle

### Q10. What is the difference between `register()` and `boot()` in a service provider?

**Answer:**

- `register()`:
  - Used to **bind things into the container**.
  - Should not depend on other services being fully booted.
  - No heavy logic, just registrations.

- `boot()`:
  - Called **after all providers are registered**.
  - Safe to use other services from the container.
  - Used for:
    - Event listener registration.
    - Route macros.
    - View composers.
    - Config tweaks that depend on other services.

Best practice:

- **All bindings** go in `register()`.
- **Side effects / initialization logic** goes in `boot()`.

---

### Q11. When would you create a custom service provider vs just relying on auto-discovery?

**Answer:**

Create a **custom service provider** when:

- You need a clean place to:
  - Bind complex services.
  - Register singletons/scoped services.
  - Register event listeners, macros, view composers.

- Your module (e.g., “Billing”, “Trading”, “Reporting”) has a **cluster of bindings** that should be grouped.

- You’re building a **package**:
  - Service provider is the main integration point to bind package services, routes, views, configs, etc.

Auto-discovery is nice, but a well-structured big app often has **domain-specific providers** like `BillingServiceProvider`, `TradingServiceProvider`, etc.

---

## 4. DI vs Facades vs Helpers

### Q12. Compare using the container (DI) vs Facades in Laravel. When do you prefer each?

**Answer:**

**Facades**:

- Pros:
  - Very concise: `Cache::get('key')`.
  - Good for framework-level services widely used across the app.
- Cons:
  - Hidden dependency; class doesn’t show what it uses from the outside.
  - Harder to swap implementations per context.
  - Slightly harder to unit test in isolation (though you can use facade fakes).

**Constructor DI / Container**:

- Pros:
  - Dependencies are **explicit** in the constructor.
  - Easy to **swap implementations** (interface bindings, contextual bindings).
  - Very testable (inject mock/fake in tests).
- Cons:
  - Slightly more verbose.

**Rule of thumb (senior-level answer):**

- For **business logic**, prefer **constructor DI** with interfaces.
- For occasional use of **framework utilities** (logging, cache, config), facades are acceptable.
- Avoid mixing too many facades deep inside core domain services if you care about portability/testability.

---

### Q13. How would you refactor a controller that does `new SomeService()` directly?

**Answer:**

Goal: avoid **hard-coded constructions** and use DI:

**Before:**

```php
class UserController extends Controller
{
    public function store(Request $request)
    {
        $service = new UserRegistrationService();
        $service->register($request->all());
    }
}
```

**After (constructor DI):**

```php
class UserController extends Controller
{
    public function __construct(
        private UserRegistrationService $registrationService,
    ) {}

    public function store(Request $request)
    {
        $this->registrationService->register($request->all());
    }
}
```

If later we want an interface:

```php
interface UserRegistrationInterface { ... }

// binding
$this->app->bind(UserRegistrationInterface::class, UserRegistrationService::class);

// controller
public function __construct(
    private UserRegistrationInterface $registrationService,
) {}
```

Now you can replace the service with a fake or different implementation without changing the controller.

---

## 5. Container Usage Patterns

### Q14. How do you manually resolve something from the container, and when is it appropriate?

**Answer:**

You can manually resolve using:

```php
$foo = app(Foo::class);
// or
$foo = resolve(Foo::class);
// or
$foo = $this->app->make(Foo::class);
```

Use manual resolution:

- In **bootstrap code** where constructor DI isn’t easily available.
- In **factories or static helpers** that need a service instance.
- In **artisan commands** outside of constructor injection (though you can also inject via constructor).

But:

- For regular services/controllers/jobs, **prefer constructor injection**.
- Too much `app()->make()` everywhere is a code smell.

---

### Q15. What is `Container::call()` (or `app()->call()`) and where do you use it?

**Answer:**

`Container::call()` allows you to **invoke a callable** (closure or method string) and automatically **inject dependencies into its parameters**.

Example:

```php
app()->call([SomeListener::class, 'handle'], [
    'event' => $event,
]);
```

- Laravel resolves `SomeListener` from the container.
- Injects any other type-hinted dependencies into `handle()`.

Use cases:

- Manually dispatching **listeners/handlers** with DI.
- Implementing **pipelines** or **command bus** patterns.
- Invoking **callables configured in config** files, where you still want DI.

---

### Q16. How does the container work with jobs, listeners, middleware, and commands?

**Answer:**

All these are **resolved via the container**:

- **Jobs**:
  - When queued job is processed, Laravel resolves the job class from the container, so constructor dependencies are injected.

- **Listeners**:
  - Event listeners can have dependencies in their constructor; the container resolves them when firing events.

- **Middleware**:
  - When middleware runs, Laravel resolves it via the container, injecting dependencies.

- **Commands**:
  - Artisan commands (extending `Command`) can receive dependencies in the constructor.

This gives a **consistent DI story** across the entire framework.

---

## 6. Design & Testing

### Q17. How does DI via the container improve testability? Give a concrete example.

**Answer:**

Because dependencies are **injected**, you can replace them with **fakes/mocks** in tests.

Example:

```php
interface SmsSender { public function send(string $phone, string $msg): void; }

class TwilioSmsSender implements SmsSender { ... }

class TwoFactorService
{
    public function __construct(private SmsSender $smsSender) {}

    public function sendCode(string $phone): void
    {
        $code = rand(100000, 999999);
        $this->smsSender->send($phone, "Code: {$code}");
    }
}
```

**Test:**

```php
class FakeSmsSender implements SmsSender {
    public array $messages = [];

    public function send(string $phone, string $msg): void
    {
        $this->messages[] = compact('phone', 'msg');
    }
}

public function test_code_is_sent_via_sms()
{
    $fake = new FakeSmsSender();

    $this->app->instance(SmsSender::class, $fake);

    $service = $this->app->make(TwoFactorService::class);

    $service->sendCode('01700000000');

    $this->assertCount(1, $fake->messages);
}
```

Because everything goes through the container, swapping `SmsSender` is trivial.

---

### Q18. How would you design DI bindings for multiple payment gateways (Stripe, PayPal, Bkash) with environment/tenant-based behavior?

**Answer (high-level design):**

- Define an interface:

```php
interface PaymentGateway
{
    public function charge(int $amount, array $meta = []): string;
}
```

- Create implementations: `StripeGateway`, `PaypalGateway`, `BkashGateway`.

- Bind using a **factory** or contextual binding:

```php
$this->app->bind(PaymentGateway::class, function ($app) {
    $gateway = config('payments.default'); // 'stripe', 'paypal', 'bkash'

    return match ($gateway) {
        'stripe' => $app->make(\App\Services\StripeGateway::class),
        'paypal' => $app->make(\App\Services\PaypalGateway::class),
        'bkash'  => $app->make(\App\Services\BkashGateway::class),
    };
});
```

- For **multi-tenant** behavior, you can read tenant config from a scoped service or request, or use **contextual binding** (`when()->needs()->give()`).

This is a common scenario interviewers like, because it shows you understand:

- Interfaces + DI.
- Container bindings.
- Keeping controllers completely agnostic of the concrete gateway.

---

### Q19. What are common pitfalls with the service container and DI in Laravel, and how do you avoid them?

**Answer:**

Some common pitfalls:

1. **God services**:
   - One huge service injected everywhere.
   - Hard to understand, hard to test.
   - Fix: split into smaller services with clear boundaries.

2. **Circular dependencies**:
   - `A` depends on `B`, `B` depends on `A` → container can’t resolve.
   - Fix: introduce a third service, events, or interfaces to break the cycle.

3. **Heavy work in service providers (`register()` / `boot()`)**:
   - Doing DB queries or remote calls in providers slows boot time.
   - Fix: keep providers mostly for binding; lazy-load heavy stuff.

4. **Leaking state with singletons under Octane/queue workers**:
   - Long-lived workers + singletons with mutable state → weird bugs across requests.
   - Fix: use **stateless singletons** or **scoped bindings** for per-request state.

5. **Overusing `app()->make()` everywhere**:
   - Hidden dependencies, harder to test.
   - Fix: prefer **constructor injection**; use manual resolution only where necessary.
