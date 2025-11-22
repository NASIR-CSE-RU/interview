# Laravel Senior Interview – OOP, SOLID & Meta-Patterns Q&A

This document focuses on **fundamentals before design patterns** – the “meta-patterns” you’ll be asked about a lot in a Laravel Senior Developer interview:

- OOP basics (encapsulation, inheritance, polymorphism, composition vs inheritance)
- SOLID principles (SRP, OCP, LSP, ISP, DIP)
- DRY, KISS, YAGNI
- Composition over inheritance

Use these questions & answers to practice explaining clearly and with Laravel/PHP-oriented examples.

---

## 1. OOP Basics

### Q1. What is encapsulation and how do you apply it in Laravel?

**Answer:**  
Encapsulation means **hiding internal details** of an object and exposing only what’s necessary through a clear interface. It keeps data and behavior that operate on that data together, and protects invariants.

In Laravel/PHP:

- Use `private` / `protected` properties and **public methods** to control access.
- Don’t expose model internals or raw arrays everywhere; expose **methods** that express intent.

Example:

```php
class Order
{
    private float $totalAmount = 0;

    public function addItem(float $price, int $qty): void
    {
        if ($qty <= 0) {
            throw new InvalidArgumentException('Qty must be positive');
        }

        $this->totalAmount += $price * $qty;
    }

    public function total(): float
    {
        return $this->totalAmount;
    }
}
```

Here, `totalAmount` is encapsulated. Outside code can’t set negative totals directly; they must use `addItem()` which enforces rules.


---

### Q2. What is inheritance? When is it useful in Laravel?

**Answer:**  
Inheritance allows a class to **extend** another class, reusing and specializing behavior.

In Laravel:

- Controllers extend `App\Http\Controllers\Controller`.
- Eloquent models extend `Illuminate\Database\Eloquent\Model`.
- Custom exceptions may extend `\Exception` or a base domain exception.

It’s useful when there is a **clear “is-a” relationship** and you want to reuse base behavior. Example: `AdminController` **is a** controller, `User` **is an** Eloquent model.


---

### Q3. What is polymorphism? Give an example in Laravel.

**Answer:**  
Polymorphism means **different classes can be treated through a common interface** and respond differently to the same method call.

Example using an interface for notifications:

```php
interface Notifier
{
    public function send(string $to, string $message): void;
}

class EmailNotifier implements Notifier
{
    public function send(string $to, string $message): void
    {
        // send email
    }
}

class SmsNotifier implements Notifier
{
    public function send(string $to, string $message): void
    {
        // send SMS
    }
}
```

In a Laravel service:

```php
class UserService
{
    public function __construct(private Notifier $notifier) {}

    public function welcomeUser(User $user): void
    {
        $this->notifier->send($user->email, 'Welcome!');
    }
}
```

Here `Notifier` is injected; at runtime it can be `EmailNotifier` or `SmsNotifier`. The service uses polymorphism and doesn’t care about the concrete implementation.


---

### Q4. What’s the difference between composition and inheritance?

**Answer:**  

- **Inheritance**: class A **is a** B (extends). It gets all behavior from base class and can override.
- **Composition**: class A **has a** B (uses). It holds references to other objects and delegates behavior.

In Laravel:

- Inheritance: `class User extends Model` – User **is a** model.
- Composition: a `UserService` **has a** `Notifier`, `UserRepository`, `Logger`, etc. via constructor injection.

**Rule of thumb**: prefer **composition** when you want flexibility and low coupling; use inheritance only when the “is-a” relationship is really strong and stable.


---

### Q5. Why is “composition over inheritance” recommended?

**Answer:**  
Because inheritance:

- Creates **tight coupling** between child and parent.
- Makes future changes riskier (changing the base can break many children).
- Can lead to deep hierarchies that are hard to reason about.

Composition:

- Encourages **small, focused classes**.
- Allows **swapping dependencies** easily (e.g. different payment gateway).
- Plays well with **interfaces and DI containers** (like Laravel’s service container).

In Laravel, you typically design your domain logic using composition: services composed of repositories, external API clients, not huge inheritance trees.


---

## 2. SOLID Principles

### Q6. Explain the Single Responsibility Principle (SRP) with a Laravel example.

**Answer:**  
SRP: A class should have **one reason to change**. It should do **one thing well**.

Bad example (violates SRP):

```php
class OrderController extends Controller
{
    public function store(Request $request)
    {
        // validate
        // calculate discount
        // talk to payment gateway
        // send emails
        // log activity
    }
}
```

Better design:

- `OrderController` – handles HTTP and delegates.
- `OrderService` – handles order creation business logic.
- `PaymentService` – handles payment gateway logic.
- `NotificationService` – sends emails/SMS.

Each class has a clearer responsibility, easier to test and change.


---

### Q7. What is the Open/Closed Principle (OCP)? How can you apply it in Laravel?

**Answer:**  
OCP: **Open for extension, closed for modification.** You should be able to add new behavior without changing existing code too much.

Example: multiple payment gateways.

Bad approach:

```php
class PaymentService
{
    public function charge(string $gateway, int $amount)
    {
        if ($gateway === 'stripe') {
            // Stripe logic
        } elseif ($gateway === 'paypal') {
            // PayPal logic
        } elseif ($gateway === 'bkash') {
            // Bkash logic
        }
    }
}
```

Every new gateway changes this class.

OCP-friendly approach:

```php
interface PaymentGateway
{
    public function charge(int $amount): string;
}

class StripeGateway implements PaymentGateway { /* ... */ }
class PaypalGateway implements PaymentGateway { /* ... */ }
class BkashGateway implements PaymentGateway { /* ... */ }

class PaymentService
{
    public function __construct(private PaymentGateway $gateway) {}

    public function charge(int $amount): string
    {
        return $this->gateway->charge($amount);
    }
}
```

Now adding a new gateway means **adding a new class** and a new binding in the service container, not modifying `PaymentService`. That’s OCP.


---

### Q8. What is the Liskov Substitution Principle (LSP)? Give a PHP example.

**Answer:**  
LSP: Objects of a superclass should be **replaceable** with objects of a subclass **without breaking correctness**. In other words, if `B` extends `A`, anywhere you expect `A` you can use `B` without surprises.

Violation example:

```php
class Rectangle
{
    protected int $width;
    protected int $height;

    public function setWidth(int $w) { $this->width = $w; }
    public function setHeight(int $h) { $this->height = $h; }
}

class Square extends Rectangle
{
    public function setWidth(int $w) {
        $this->width = $w;
        $this->height = $w; // surprise
    }

    public function setHeight(int $h) {
        $this->height = $h;
        $this->width = $h; // surprise
    }
}
```

Code that expects a `Rectangle` but gets a `Square` will behave strangely when setting width and height independently – **LSP is broken**.

In Laravel, violating LSP usually happens when a subclass **throws exceptions or ignores methods** in ways that the base class’s contract doesn’t allow. Keep your subclasses faithful to the base class behavior.


---

### Q9. What is the Interface Segregation Principle (ISP)?

**Answer:**  
ISP: **Clients should not be forced to depend on methods they do not use.**

Instead of one “god interface” with many methods, create smaller, focused interfaces.

Bad example:

```php
interface ReportInterface
{
    public function generateHtml();
    public function generatePdf();
    public function generateCsv();
}
```

If some implementations only support HTML, they are forced to implement methods they don’t need.

Better:

```php
interface HtmlReport { public function generateHtml(); }
interface PdfReport { public function generatePdf(); }
interface CsvReport { public function generateCsv(); }
```

In Laravel, you might create separate interfaces for:

- `Cacheable`
- `ExportableToCsv`
- `ExportableToPdf`

so classes implement only what they actually support.


---

### Q10. What is the Dependency Inversion Principle (DIP) and how does Laravel’s service container help with it?

**Answer:**  
DIP has two parts:

1. **High-level modules** should not depend on low-level modules; both should depend on **abstractions** (interfaces).
2. **Abstractions should not depend on details; details should depend on abstractions.**

In Laravel:

- You define interfaces (e.g. `PaymentGateway`, `UserRepository`) and **type-hint them** in constructors.
- Service container binds interfaces to concrete classes:

```php
$this->app->bind(PaymentGateway::class, StripeGateway::class);
```

Controllers and services depend on `PaymentGateway` (abstraction), not on `StripeGateway` (detail). This makes code **more flexible and testable**, because you can swap concrete implementations easily, including mocks for tests.


---

## 3. DRY, KISS, YAGNI & Composition Over Inheritance

### Q11. What does DRY mean? Give a Laravel example.

**Answer:**  
DRY = **Don’t Repeat Yourself**.

It means avoid duplicating **knowledge and logic** in multiple places. If business rules change, you should change them in **one place**, not many.

Example:

- If you validate an email format, uniqueness, and domain in multiple controllers, that’s repetition.
- Better: extract a `FormRequest` or **service** (or custom validation rule) that contains the logic and reuse it everywhere.

```php
class StoreUserRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'email' => ['required', 'email', 'unique:users,email'],
        ];
    }
}
```

Now the user validation logic lives in one place.


---

### Q12. What is the KISS principle and how do you apply it in Laravel projects?

**Answer:**  
KISS = **Keep It Simple, Stupid** (or “Keep It Simple and Straightforward”).

- Don’t over-engineer solutions.
- Prefer **simple, readable code** over clever but complex tricks.
- Avoid unnecessary abstractions and indirection.

In Laravel:

- Don’t introduce 10 layers of services and factories for a simple CRUD.
- Use built-in features (validation, Eloquent, queues) before building complex custom frameworks.
- When explaining to a junior dev, the flow should be understandable: Controller → Service → Repository (if needed) → Model.


---

### Q13. What is YAGNI and why is it important in a Laravel codebase?

**Answer:**  
YAGNI = **You Aren’t Gonna Need It**.

It warns against implementing features or abstractions **before** they’re actually needed.

In Laravel:

- Don’t create 15 repository interfaces when your project has only 3 simple CRUD endpoints.
- Don’t build a complex plugin system when you only have one payment gateway.

YAGNI helps keep the codebase **lean**, reduces maintenance cost, and avoids unused abstractions that confuse the team.


---

### Q14. Can you give a concrete example of “composition over inheritance” in a Laravel backend?

**Answer:**  
Instead of:

```php
class BaseService
{
    public function log($message) { /* ... */ }
    public function dispatchJob($job) { /* ... */ }
}

class OrderService extends BaseService
{
    public function createOrder(array $data) { /* ... */ }
}
```

Here, `OrderService` is tightly coupled to `BaseService`.

Use composition:

```php
class OrderService
{
    public function __construct(
        private LoggerInterface $logger,
        private Dispatcher $dispatcher,
    ) {}

    public function createOrder(array $data)
    {
        // ...
        $this->logger->info('Order created');
        $this->dispatcher->dispatch(new SendOrderEmailJob($data['user_id']));
    }
}
```

Now `OrderService` **has** a logger and dispatcher instead of **being** a `BaseService`. This is **composition over inheritance**, more flexible and testable.


---

### Q15. How do these meta-patterns (OOP, SOLID, DRY/KISS/YAGNI) show up in a real Laravel senior-level code review?

**Answer:**  
As a senior, you’re expected to:

- Spot **God classes** and huge controllers that violate SRP.
- Encourage **interfaces + DI** instead of hard-coded concretions (DIP, polymorphism).
- Avoid duplicated logic by extracting **services, form requests, validation rules** (DRY).
- Push back when someone wants premature abstractions or microservices for a tiny app (KISS, YAGNI).
- Prefer **composition** of small services over deep inheritance trees.

In a review, you might say things like:

- “Let’s move this logic from the controller into a dedicated `OrderService` (SRP).”
- “Instead of checking `$gateway === 'stripe'`, can we introduce a `PaymentGateway` interface (OCP, DIP)?”
- “We’re reusing this validation logic in three places, let’s extract a `FormRequest` (DRY).”
- “This inheritance is unnecessary; we can inject the collaborator class instead (composition over inheritance).”


---

### Q16. If you had to explain SOLID to a non-technical product manager in one minute, how would you do it?

**Answer (short version):**  

- **Single Responsibility:** each part of the system should do **one job well**, so it’s easier to fix/extend.
- **Open/Closed:** we can **add new features** without breaking old ones every time.
- **Liskov Substitution:** when we say “this behaves like X,” it really behaves like X; no surprises.
- **Interface Segregation:** we give each component **only the options it needs**, so it stays simple.
- **Dependency Inversion:** the high-level logic depends on **interfaces/contracts**, so we can change details (e.g. payment provider) without rewriting business logic.

This keeps the system **easier to change**, **safer to refactor**, and more reliable as it grows.


---

### Q17. How would you coach a junior Laravel dev to start applying these principles?

**Answer:**  

- Start with **SRP and DRY**: move logic out of controllers, avoid copy-paste.
- Introduce **interfaces + DI** gradually: create one or two interfaces for external services (e.g., payment, notifications).
- Use **code reviews** to highlight simple KISS/YAGNI wins: “We don’t need a complex pattern here, just a small service class.”
- Show concrete before/after examples where refactoring improved readability and testability.
- Keep explanations short and practical, focusing on **real Laravel code**, not just theory.

Over time, they’ll naturally learn to think in SOLID terms and use composition over inheritance.


---

_End of file._
