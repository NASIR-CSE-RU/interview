# Laravel Senior Interview – “Patterns in Practice” Q&A

This file covers **Laravel-specific patterns** that aren’t always formal GoF names but show up in real-world projects and senior interviews:

- Repository + Service + Controller layering
- Form Request classes (validation pattern)
- Policy / Gate pattern for authorization
- Action classes (one class per use-case)
- Pipelines (`Illuminate\Pipeline`)
- Model Observer classes
- Resource / Transformer pattern (API resources)
- Specification-like logic for complex filters

Use this as a practical prep sheet: questions are phrased like an interviewer; answers show how you’d explain them with Laravel context.

---

## 1. Repository + Service + Controller Layering

### Q1. Can you describe a typical Repository + Service + Controller layering in a Laravel project? Why use it?

**Answer:**  
A common layering is:

- **Controller** – handles HTTP concerns only: request/response, status codes, resources.
- **Service / Use-case** – contains **business logic** and coordinates repositories, external APIs, events.
- **Repository** – encapsulates **data access** (Eloquent/DB queries).

Typical flow:

1. Controller receives request and calls a **Form Request** for validation.
2. Controller calls a **Service** method like `CreateOrderService::handle($data)`.
3. Service uses one or more **Repositories** to read/write from DB and maybe calls external services.
4. Controller wraps the result in a **Resource** and returns JSON.

Benefits: clear separation of concerns, easier testing, and the ability to swap data sources or logic without touching controllers everywhere.

---

### Q2. When would you *not* introduce a Service and Repository layer for a Laravel feature?

**Answer:**  
For **very simple CRUD** or small modules, adding both Services and Repositories can be overkill:

- If your controller only does `Post::create($request->validated())` and `Post::paginate()`, a full-blown `PostRepository` + `PostService` may not add real value.
- Laravel already gives Eloquent as a rich data layer, so you can start with **Controller + Model** and only introduce extra layers when:
  - Business rules become complex.
  - You need to integrate multiple models/APIs.
  - You feel pain in unit testing or code reuse.

A senior engineer should be able to justify **when** extra layers reduce complexity vs. when they add noise.

---

## 2. Form Request Classes – Validation Pattern

### Q3. What is a Form Request in Laravel and why is it considered a “validation pattern”?

**Answer:**  
A **Form Request** is a custom request class extending `Illuminate\Foundation\Http\FormRequest` that holds:

- Validation rules (`rules()`)
- Authorization logic (`authorize()`)
- Optional custom messages / attributes

It acts as a **pattern for validation** because:

- It moves validation rules out of controllers, keeping them clean.
- It groups **validation + authorize** logic for a specific endpoint in one place.
- It is reusable and easily testable.

Example:

```php
class StoreOrderRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'items' => ['required', 'array', 'min:1'],
            'items.*.id' => ['required', 'integer', 'exists:products,id'],
        ];
    }
}
```

Controller signature:

```php
public function store(StoreOrderRequest $request, CreateOrderService $service)
{
    $order = $service->handle($request->validated());
    return new OrderResource($order);
}
```

---

### Q4. How do Form Requests help with reusability and consistency in a large Laravel codebase?

**Answer:**  

- Same **validation rules** can be reused across multiple controllers/routes.
- You avoid duplicating complex rule arrays (`required_if`, `sometimes`, custom `Rule::when`) everywhere.
- If business rules change (e.g., minimum quantity), you update **only** the Form Request, not dozens of controllers.

In an interview, you can say: “Form Requests are our standard pattern for validation + authorization per endpoint; they centralize rules and keep controllers focused on orchestration.”

---

## 3. Policy / Gate Pattern for Authorization

### Q5. Explain the difference between Gates and Policies in Laravel. How do they form an authorization pattern?

**Answer:**  

- **Gates**: closures that define authorization rules for general abilities: `Gate::define('view-report', fn ($user) => ...)`. They’re good for simple, **global** checks.
- **Policies**: classes that group authorization logic for a specific model, e.g., `UserPolicy`, `OrderPolicy` with methods `view`, `update`, `delete`.

Together they form an **authorization pattern**:

- Instead of sprinkling `if ($user->id === $order->user_id)` everywhere, you centralize it in policies.
- Controllers and views use expressive methods: `$this->authorize('update', $order)` or `@can('update', $order)`.

This keeps access control consistent, testable, and easy to change in one place.

---

### Q6. How would you handle multi-tenant authorization using Policies in Laravel?

**Answer:**  
In a multi-tenant app, your policy methods can enforce tenant boundaries:

```php
class OrderPolicy
{
    public function view(User $user, Order $order): bool
    {
        return $user->tenant_id === $order->tenant_id;
    }
}
```

You might also:

- Use **global scopes** or custom middlewares to ensure all queries are tenant-scoped.
- Combine **tenant checks** inside policies for critical actions (update/delete).

So you can explain: “We use Policies as the standard pattern to enforce both role-based and tenant-based rules consistently.”

---

## 4. Action Classes – One Class per Use Case

### Q7. What are Action classes (e.g. `CreateOrderAction`) and why might you use them over a fat Service class?

**Answer:**  
Action classes are **small, focused classes**, usually one per use-case, like:

- `CreateOrderAction`
- `CancelOrderAction`
- `AssignDriverToRideAction`

Each has a single public `__invoke()` or `handle()` method:

```php
class CreateOrderAction
{
    public function __construct(
        private OrderRepository $orders,
        private PaymentGateway $payments,
    ) {}

    public function __invoke(array $data): Order
    {
        // business logic
    }
}
```

Benefits vs. one big `OrderService`:

- Better adherence to **Single Responsibility**.
- Easier to test and reason about each use-case.
- You can wire them into controllers, jobs, console commands, and pipelines easily.

---

### Q8. How do Action classes help with reusability across HTTP, CLI, and queues?

**Answer:**  
Because Actions encapsulate **pure business operations**, you can reuse them in different entry points:

- HTTP controller: `CreateOrderAction` via route.
- CLI command: `php artisan orders:create ...` calling the same Action.
- Queue job: background processing also calls the Action.

All entry points share the same **core behavior**, so you avoid duplicating logic in multiple places.

---

## 5. Pipelines – `Illuminate\Pipeline`

### Q9. What is the Pipeline pattern in Laravel and when would you use it?

**Answer:**  
The **Pipeline** in Laravel (`Illuminate\Pipeline\Pipeline`) implements a pattern similar to **Chain of Responsibility**:

- You send an object (payload) through a series of **pipes** (classes/closures).
- Each pipe can transform the payload and pass it to the next.

Example usage: processing an order through multiple steps (validate stock → apply discounts → calculate totals → persist).

```php
use Illuminate\Pipeline\Pipeline;

$processed = app(Pipeline::class)
    ->send($orderData)
    ->through([
        CheckStock::class,
        ApplyDiscounts::class,
        CalculateTotals::class,
    ])
    ->thenReturn();
```

It keeps each step **small and composable**. You can easily re-order or disable steps via configuration.

---

### Q10. How is the Pipeline pattern different from a big Service method with many if/else blocks?

**Answer:**  

- In a big Service method, all steps are **in one place** and tightly coupled; hard to test individually and reorder.
- With Pipelines:
  - Each step is a **separate class** with a single responsibility.
  - You can **unit test** each pipe in isolation.
  - You can **dynamically configure** which pipes run based on environment or tenant needs.

It’s a good answer to: “How do you keep complex workflows flexible and maintainable?”

---

## 6. Model Observer Classes

### Q11. What are Model Observers in Laravel and why are they considered a pattern?

**Answer:**  
Model Observers are classes that group **callbacks for model events** like `creating`, `created`, `updating`, `deleted`, etc.

Example:

```php
class UserObserver
{
    public function created(User $user)
    {
        // Send welcome email, log activity, etc.
    }
}
```

Registering:

```php
User::observe(UserObserver::class);
```

This is a pattern because it:

- Centralizes side-effects related to model lifecycle.
- Keeps models and controllers **cleaner**.
- Encourages **event-driven** thinking (reacting to changes).

In an interview, mention that you avoid putting heavy logic in model events directly (`User::creating(function () { ... })`) and instead use Observers for maintainability.

---

## 7. Resource / Transformer Pattern (API Resources)

### Q12. What problem do Laravel API Resources solve? How is this a “transformer pattern”?

**Answer:**  
Laravel API Resources (classes extending `JsonResource`) act as **transformers** between models and API responses. They:

- Control exactly what fields are exposed in the API.
- Allow **shaping** responses (renaming, nesting, adding computed fields).
- Keep presentation logic out of controllers and models.

Example:

```php
class OrderResource extends JsonResource
{
    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'total' => $this->total_amount,
            'status' => $this->status,
            'placed_at' => $this->created_at->toIso8601String(),
            'customer' => new UserResource($this->whenLoaded('user')),
        ];
    }
}
```

This is effectively the **Resource / Transformer pattern**: turning domain models into API-friendly JSON structures.

---

### Q13. How do Resources help with versioning and backward compatibility in APIs?

**Answer:**  
Because Resources centralize the **output shape**, you can easily:

- Introduce `v2` resources with new shapes while keeping `v1` unchanged.
- Conditionally include fields based on request headers or version params.
- Deprecate fields gradually by controlling them in the Resource layer.

This means internal model changes don’t automatically break API contracts – you control the output **at one layer**.

---

## 8. Specification-like Logic for Complex Filters

### Q14. What is the “Specification-like” pattern for query filters in Laravel?

**Answer:**  
The idea is to model **filter conditions** as reusable objects or classes, similar to the **Specification pattern**.

Instead of big `if (!empty($request->status)) { ... }` chains in controllers, you create small filter classes or use a filter pipeline.

Simple class-based approach:

```php
interface Filter
{
    public function apply(Builder $query, $value): Builder;
}

class StatusFilter implements Filter
{
    public function apply(Builder $query, $value): Builder
    {
        return $query->where('status', $value);
    }
}
```

You then have a `OrderFilters` class that applies all filters based on request data.

This keeps complex filtering logic **composable and testable**.


### Q15. Can you describe a practical way to implement request-based filters using a “filters” pattern in Laravel?

**Answer:**  
One practical way:

1. Define a mapping of **query params → filter classes**:

```php
class OrderFilters
{
    protected array $filters = [
        'status' => StatusFilter::class,
        'from' => FromDateFilter::class,
        'to' => ToDateFilter::class,
        'customer' => CustomerFilter::class,
    ];

    public function apply(Builder $query, Request $request): Builder
    {
        foreach ($this->filters as $param => $filterClass) {
            if ($request->filled($param)) {
                $filter = app($filterClass);
                $query = $filter->apply($query, $request->input($param));
            }
        }

        return $query;
    }
}
```

2. Use in repository or controller:

```php
public function index(Request $request, OrderFilters $filters)
{
    $query = Order::query();
    $query = $filters->apply($query, $request);

    return OrderResource::collection($query->paginate());
}
```

This is “Specification-like”: each filter encapsulates a **piece of criteria**, and `OrderFilters` composes them based on input.

---

## 9. Putting It All Together

### Q16. How would you describe your overall architectural style in a senior Laravel interview using these patterns?

**Answer (summary style):**  

> “I usually organize a Laravel backend as a set of layers and patterns:
> - Controllers stay thin and delegate to **Action/Service classes**.
> - Validation and basic authorization go into **Form Requests**.
> - Data access is encapsulated in **Repositories** where needed.
> - Authorization logic lives in **Policies/Gates**, and complex workflows use **Pipelines**.
> - Side-effects around models go into **Observers**, and all API responses are shaped through **Resources/Transformers**.
> - For complex searching, I like a **filters/specification-like pattern** so each condition is testable and composable.
> 
> This combination keeps the codebase modular, testable, and easy to evolve as business requirements change.”

That kind of high-level explanation shows you understand how these patterns work **together**, not just individually.

---

_End of file._
