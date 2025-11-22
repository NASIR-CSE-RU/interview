# Laravel Senior Interview – Design Patterns Q&A

This document focuses on **common design patterns in Laravel** you’ll be asked about as a senior backend engineer:

- Repository Pattern
- Service / Use-Case Layer
- Strategy Pattern
- Factory Pattern
- Observer / Events & Listeners
- Adapter Pattern
- Decorator Pattern (incl. Middleware)
- Facade Pattern (and Laravel Facades)
- CQRS-style separation
- When NOT to use patterns / over-engineering

Use this as an interview prep sheet and to practice your explanations with Laravel-specific examples.

---

## 1. General

### Q1. Why do we use design patterns in a Laravel application?

**Answer:**  
Design patterns are **reusable solutions** to common design problems. In Laravel they help you:

- Keep code **modular and testable**.
- Separate concerns (domain logic vs infrastructure vs delivery).
- Make it easier to **swap implementations** (e.g., payment gateways, storage providers).
- Improve **team communication** – saying “use a Strategy here” is faster than explaining from scratch.

But patterns are a **means, not a goal**. You introduce them when they make the code clearer, not just to sound “enterprise”.

---

## 2. Repository Pattern

### Q2. What is the Repository pattern and why might you use it in Laravel?

**Answer:**  
A Repository is a **mediator between your domain/business logic and the data layer**. It hides how data is stored (Eloquent, raw SQL, external API) and exposes a clean interface:

```php
interface UserRepository
{
    public function findById(int $id): ?User;
    public function findByEmail(string $email): ?User;
    public function create(array $data): User;
}
```

Implementation using Eloquent:

```php
class EloquentUserRepository implements UserRepository
{
    public function findById(int $id): ?User
    {
        return User::find($id);
    }

    public function findByEmail(string $email): ?User
    {
        return User::where('email', $email)->first();
    }

    public function create(array $data): User
    {
        return User::create($data);
    }
}
```

Benefits:

- **Testability**: you can mock `UserRepository` in tests.
- **Flexibility**: you can swap Eloquent with another storage (e.g., external user service) without changing the service layer.
- **Separation of concerns**: controllers/services don’t deal with Eloquent directly.

### Q3. When is the Repository pattern overkill in Laravel?

**Answer:**  
For **very simple CRUD** modules or small apps, repositories can add unnecessary abstraction:

- If your controllers just do `User::create()` and `User::paginate()`, adding `UserRepository` may not add real value.
- Repositories shine when you have **complex queries, multiple data sources, or you need strict boundaries** around the domain.

Rule: start simple; introduce repositories when you feel pain in tests or when swapping implementations.


---

## 3. Service / Use-Case Layer

### Q4. What is a Service (or Use-Case) layer in Laravel and why use it?

**Answer:**  
A Service/Use-Case class encapsulates a **piece of business logic** (a use-case) so controllers stay thin and focused on HTTP concerns:

```php
class CreateOrderService
{
    public function __construct(
        private OrderRepository $orders,
        private PaymentGateway $payments,
        private Notifier $notifier,
    ) {}

    public function handle(array $data): Order
    {
        // validate domain rules, create order, charge, send notification
    }
}
```

Controller:

```php
class OrderController extends Controller
{
    public function store(Request $request, CreateOrderService $service)
    {
        $order = $service->handle($request->validated());
        return new OrderResource($order);
    }
}
```

Benefits:

- **SRP** for controllers.
- Reusable **use-cases** from API, CLI, jobs, etc.
- Easier to test business logic in isolation.


---

## 4. Strategy Pattern

### Q5. Explain the Strategy pattern with a Laravel example (e.g., multiple payment gateways).

**Answer:**  
Strategy lets you **pick an algorithm at runtime** via a common interface.

Interface:

```php
interface PaymentGateway
{
    public function charge(int $amount, array $meta = []): string;
}
```

Concrete strategies:

```php
class StripeGateway implements PaymentGateway
{
    public function charge(int $amount, array $meta = []): string { /* ... */ }
}

class PaypalGateway implements PaymentGateway
{
    public function charge(int $amount, array $meta = []): string { /* ... */ }
}

class BkashGateway implements PaymentGateway
{
    public function charge(int $amount, array $meta = []): string { /* ... */ }
}
```

Context class:

```php
class PaymentService
{
    public function __construct(private PaymentGateway $gateway) {}

    public function chargeOrder(Order $order): string
    {
        return $this->gateway->charge($order->amount, ['order_id' => $order->id]);
    }
}
```

Binding in `AppServiceProvider`:

```php
$this->app->bind(PaymentGateway::class, function ($app) {
    return match (config('payments.default')) {
        'stripe' => $app->make(StripeGateway::class),
        'paypal' => $app->make(PaypalGateway::class),
        'bkash'  => $app->make(BkashGateway::class),
        default  => $app->make(StripeGateway::class),
    };
});
```

Now you can change the payment strategy via config or environment without changing `PaymentService`.


### Q6. How would you make the Strategy pattern tenant-based or user-based?

**Answer:**  
Instead of binding a single default implementation, you can:

- Create a **factory service** that looks at tenant/user configuration and returns the right strategy; or
- Use **contextual binding** in the service container.

Factory example:

```php
class PaymentGatewayFactory
{
    public function __construct(
        private StripeGateway $stripe,
        private PaypalGateway $paypal,
        private BkashGateway $bkash,
    ) {}

    public function forTenant(Tenant $tenant): PaymentGateway
    {
        return match ($tenant->payment_gateway) {
            'stripe' => $this->stripe,
            'paypal' => $this->paypal,
            'bkash'  => $this->bkash,
        };
    }
}
```

Your service receives the tenant and asks the factory for the right strategy.


---

## 5. Factory Pattern

### Q7. What is the Factory pattern? Give an example from Laravel or your own code.

**Answer:**  
A Factory is responsible for **creating objects**; it hides the creation logic so callers don’t need to know all the details.

In Laravel core, examples:

- `Model::factory()` for test data (Eloquent model factories).
- `Cache::store('redis')` – the cache manager is a factory.

Custom example: creating different notification senders based on config:

```php
class NotifierFactory
{
    public function __construct(
        private EmailNotifier $email,
        private SmsNotifier $sms,
    ) {}

    public function make(string $channel): Notifier
    {
        return match ($channel) {
            'email' => $this->email,
            'sms'   => $this->sms,
            default => $this->email,
        };
    }
}
```

Now your code can do `$notifier = $factory->make('sms');` without worrying about `new SmsNotifier()`.


---

## 6. Observer Pattern (Events & Listeners)

### Q8. How do you implement the Observer pattern in Laravel?

**Answer:**  
Laravel has built-in support via **Model Observers** and **Events/Listeners**.

**Model Observer example:**

```php
class OrderObserver
{
    public function created(Order $order)
    {
        // called when an order is created
        ActivityLog::create([...]);
    }
}
```

Register:

```php
Order::observe(OrderObserver::class);
```

**Events & Listeners example:**

```php
class OrderPlaced
{
    public function __construct(public Order $order) {}
}

class SendOrderConfirmation
{
    public function handle(OrderPlaced $event)
    {
        Mail::to($event->order->user)->send(new OrderConfirmationMail($event->order));
    }
}
```

In `EventServiceProvider`:

```php
protected $listen = [
    OrderPlaced::class => [
        SendOrderConfirmation::class,
        LogOrderPlaced::class,
    ],
];
```

Controller or service:

```php
event(new OrderPlaced($order));
```

This is the Observer pattern: **subjects** (Orders) emit events; **observers** (listeners) react, loosely coupled.


---

## 7. Adapter Pattern

### Q9. What is the Adapter pattern? Provide an example integrating a third-party API in Laravel.

**Answer:**  
Adapter wraps a **third-party interface** and exposes your own **clean interface** so the rest of the code doesn’t depend on the vendor’s SDK directly.

Example: integrating a 3rd-party SMS provider:

Vendor SDK:

```php
class VendorSmsClient
{
    public function sendMessage(string $phone, string $body): void { /* ... */ }
}
```

Your interface + adapter:

```php
interface SmsGateway
{
    public function send(string $phone, string $message): void;
}

class VendorSmsAdapter implements SmsGateway
{
    public function __construct(private VendorSmsClient $client) {}

    public function send(string $phone, string $message): void
    {
        $this->client->sendMessage($phone, $message);
    }
}
```

Now `UserService` or `NotificationService` depends on `SmsGateway`, not `VendorSmsClient`. If you change SMS provider, you just create another adapter.


---

## 8. Decorator Pattern (including Middleware)

### Q10. How is the Decorator pattern used in Laravel?

**Answer:**  
Decorator adds behavior **around** an object without changing its interface.

In Laravel:

- **HTTP middleware** decorates the request/response cycle.
- You can decorate services manually by wrapping them.

Example: logging decorator for `PaymentGateway`:

```php
class LoggingPaymentGateway implements PaymentGateway
{
    public function __construct(private PaymentGateway $inner) {}

    public function charge(int $amount, array $meta = []): string
    {
        \Log::info('Charging payment', ['amount' => $amount, 'meta' => $meta]);
        $result = $this->inner->charge($amount, $meta);
        \Log::info('Charge result', ['result' => $result]);

        return $result;
    }
}
```

Binding:

```php
$this->app->bind(PaymentGateway::class, function ($app) {
    $gateway = $app->make(StripeGateway::class);
    return new LoggingPaymentGateway($gateway);
});
```

You added logging **without modifying** `StripeGateway`.


### Q11. How is middleware a form of Decorator?

**Answer:**  
Each middleware receives the request and a “next” callback. It can do work **before and/or after** calling `next($request)` and eventually returning a response.

This is basically a **chain of decorators** around the core handler (controller). Each middleware decorates the handling of the request (auth, logging, rate-limiting, etc.).


---

## 9. Facade Pattern (and Laravel Facades)

### Q12. What is the Facade pattern and how do Laravel Facades relate to it?

**Answer:**  
A Facade provides a **simple, unified interface** to a complex subsystem.

Laravel Facades (like `Cache`, `DB`, `Queue`, `Auth`) are **static proxies** that hide the service container’s complexity:

```php
Cache::put('key', 'value', 600);
```

Under the hood, `Cache` facade resolves a cache manager instance from the container and calls `put` on it.

Benefits:

- **Convenient syntax**.
- Unifies access to a subsystem behind a simple class (`Cache`, `Mail`, `Log`).

But for complex domain logic, it’s usually better to use **dependency injection** instead of relying heavily on facades, especially for testability.


### Q13. When would you prefer DI over facades?

**Answer:**  

- When you want to **unit test** classes by mocking their dependencies.
- When you need multiple implementations of the same abstraction.
- When you want to make dependencies explicit and avoid hidden global state.

Example – instead of:

```php
class OrderService
{
    public function createOrder(array $data)
    {
        $order = Order::create($data);
        Mail::to($order->user)->send(new OrderMail($order));
    }
}
```

Prefer:

```php
class OrderService
{
    public function __construct(
        private Mailer $mailer,
    ) {}

    public function createOrder(array $data)
    {
        $order = Order::create($data);
        $this->mailer->to($order->user)->send(new OrderMail($order));
    }
}
```

Now in tests you can mock `Mailer`.


---

## 10. CQRS-style Separation

### Q14. What is CQRS and how can you apply a light version of it in Laravel?

**Answer:**  
CQRS (Command Query Responsibility Segregation) suggests **separating reads and writes**:

- **Commands**: change state (create order, cancel booking).
- **Queries**: read state (list orders, show dashboard).

Lightweight application in Laravel:

- Separate **service classes** or handlers for commands vs queries (`CreateOrderService`, `GetOrderListService`).
- Use **different models or query builders** for read models (e.g., using `selectRaw`, `DB` queries, or dedicated `ReadOrderRepository` for dashboards).

You don’t need full event sourcing to benefit from CQRS ideas; just separating read/write paths often improves clarity and scalability.


---

## 11. Over-Engineering & Pattern Abuse

### Q15. How do you decide when NOT to use a design pattern in Laravel?

**Answer:**  

Ask:

1. **Does this pattern actually solve a problem we have today?**  
   If not, it’s probably YAGNI.

2. **Does it make the code easier to read for the team?**  
   If it confuses juniors and doesn’t give clear benefits, skip it.

3. **Is there a simpler way using Laravel’s built-in features?**  
   For example, don’t build a custom event system when Laravel Events already do the job.

Good guideline for a senior dev:

- Start with **simple, clear code**.
- When you see repetition, complexity, or need for flexibility, introduce the **smallest pattern** that solves that pain.
- Always keep **tests** and **maintainability** in mind – patterns are tools to achieve those goals, not just buzzwords.


---

_End of file._
