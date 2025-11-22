# Laravel Eloquent – Senior Developer Interview Q&A

> Focus: Deep understanding of **Eloquent ORM** for a senior Laravel backend role.

---

## 1. Fundamentals & Model Design

### Q1. What is Eloquent, and how is it different from the Query Builder?

**Answer:**

- **Eloquent** is Laravel’s ORM that implements the **Active Record** pattern.
  - Each model class represents a table (or view).
  - Each model instance represents a row.
  - Business logic can live on the model (methods, accessors, scopes, relationships).
- **Query Builder** is a lower-level, fluent interface for building SQL queries:
  - Returns `Collection`/arrays, not model instances.
  - No relationships, accessors, events, etc.

**When to use what:**

- Use **Eloquent** for:
  - Domain logic
  - Relationships
  - CRUD on entities
- Use **Query Builder** for:
  - Heavy reporting
  - Complex aggregates
  - Performance-critical queries where you don’t need models

---

### Q2. How does Eloquent map models to database tables and keys?

**Answer:**

By convention:

- Model `User` → table `users`
- Model `OrderItem` → table `order_items`
- Primary key column: `id` (auto-increment int)
- Timestamps: `created_at`, `updated_at`

You can override:

```php
class OrderItem extends Model
{
    protected $table = 'b2c_order_items';
    protected $primaryKey = 'order_item_id';
    public $incrementing = false;
    protected $keyType = 'string';
    public $timestamps = false;
}
```

Senior-level points:

- Know when to override `$table`, `$primaryKey`, `$keyType`, `$incrementing`.
- Understand the impact of **non-incrementing / UUID** primary keys.

---

### Q3. What are accessors, mutators, and casts in Eloquent, and when do you use each?

**Answer:**

- **Casts** (`$casts`):
  - Convert column values to native types automatically.
  - Example: `protected $casts = ['meta' => 'array', 'price' => 'decimal:2'];`
  - Great for JSON columns, dates, `enum`s, value objects (with custom casts).

- **Accessors**:
  - Transform a value **when reading** from the model.
  - New syntax:
    ```php
    protected function fullName(): Attribute
    {
        return Attribute::get(fn () => "{$this->first_name} {$this->last_name}");
    }
    ```
  - Often used for **computed attributes**.

- **Mutators**:
  - Transform a value **when setting**.
  - New syntax:
    ```php
    protected function password(): Attribute
    {
        return Attribute::set(fn ($value) => bcrypt($value));
    }
    ```

Senior point: prefer **casts** for types; use accessors/mutators for **business logic** or computed values.

---

### Q4. What are local and global scopes? When would you use them?

**Answer:**

- **Local scope**: Reusable query fragment on one model.

  ```php
  public function scopeActive($query)
  {
      return $query->where('status', 'active');
  }

  // usage:
  User::active()->get();
  ```

- **Global scope**: Automatically applied to all queries of a model.

  ```php
  class TenantScope implements Scope
  {
      public function apply(Builder $builder, Model $model)
      {
          $builder->where('tenant_id', tenant()->id);
      }
  }

  class Order extends Model
  {
      protected static function booted()
      {
          static::addGlobalScope(new TenantScope);
      }
  }
  ```

- **Disabling a global scope**:

  ```php
  Order::withoutGlobalScope(TenantScope::class)->get();
  ```

Use global scopes for **multi-tenancy, soft-deletes, “only active records”**. Use local scopes for common filters like `active()`, `paid()`, `forUser()`.

---

### Q5. What is mass assignment in Eloquent, and how do you secure it?

**Answer:**

Mass assignment:

```php
User::create($request->all()); // risk!
```

Eloquent protects by requiring either:

- `$fillable` (whitelist) – recommended:
  ```php
  protected $fillable = ['name', 'email', 'password'];
  ```
- `$guarded` (blacklist) – often set to `[]` in internal models:
  ```php
  protected $guarded = []; // all attributes mass-assignable (be careful)
  ```

Security:

- Never allow mass assignment on **sensitive fields** (`is_admin`, `role`, `balance`) via user input.
- In interviews, emphasize you **prefer `$fillable`** or DTO/Request objects to control fields.

---

## 2. Relationships & Advanced Querying

### Q6. Explain Eloquent relationship types and give examples.

**Answer:**

Common relationships:

- **One to One**:
  ```php
  public function profile()
  {
      return $this->hasOne(Profile::class);
  }
  ```

- **One to Many**:
  ```php
  public function orders()
  {
      return $this->hasMany(Order::class);
  }
  ```

- **Many to Many**:
  ```php
  public function roles()
  {
      return $this->belongsToMany(Role::class)->withTimestamps();
  }
  ```

- **Polymorphic**:
  - `morphOne`, `morphMany`, `morphTo`, `morphToMany`.
  - Example: `Comment` belonging to `Post` or `Video` via `commentable_id`, `commentable_type`.

Senior-level expectations:

- You can design relationships for **permissions, tags, logs**.
- You know when **polymorphic** is appropriate vs separate tables.

---

### Q7. What are `hasOneThrough` and `hasManyThrough`? When would you use them?

**Answer:**

These let you define relationships **through an intermediate model**.

Example: Country → hasManyThrough → Post, via User:

```php
class Country extends Model
{
    public function posts()
    {
        return $this->hasManyThrough(Post::class, User::class);
    }
}
```

- Eloquent uses `users.country_id` and `posts.user_id` to fetch posts for all users in a country.
- Good for **“X through Y”** relationships without custom joins in controllers.

---

### Q8. How do you avoid N+1 query problems with Eloquent?

**Answer:**

N+1 pattern:

```php
$orders = Order::all(); // 1 query

foreach ($orders as $order) {
    echo $order->user->name; // 1 query per order => N+1
}
```

Fix via **eager loading**:

```php
$orders = Order::with('user')->get(); // 2 queries total
```

Advanced:

- Constraint eager loading:
  ```php
  Order::with(['items' => fn ($q) => $q->where('status', 'active')])->get();
  ```
- Use `load()` to lazy eager load after initial query.
- Use `withCount()` to avoid `->relation->count()` per row.

In a senior interview, mention:

- Recognize N+1 via **debugbar/Telescope/queries log**.
- Always think about **eager loading** and **selecting only needed columns**.

---

### Q9. How do you query based on relationship existence or counts?

**Answer:**

Use:

- `has()`, `whereHas()`:

  ```php
  // users with at least one post
  User::has('posts')->get();

  // users with posts in last 7 days
  User::whereHas('posts', fn ($q) =>
      $q->where('created_at', '>=', now()->subDays(7))
  )->get();
  ```

- `withCount()`:

  ```php
  $users = User::withCount('posts')->get();

  // filter by count
  User::withCount('posts')->having('posts_count', '>', 10)->get();
  ```

These are more efficient than loading the relationship first and then filtering in PHP.

---

### Q10. Explain `chunk`, `cursor`, `lazy`, and `lazyById`. When would you use each?

**Answer:**

Used for **processing large datasets** without loading everything into memory.

- `chunk($size, $callback)`:
  - Fetches **fixed-size pages** with offsets.
  - Good when stable data and you don’t care about new rows appearing.
  - Can miss/duplicate rows if data changes while iterating.

- `cursor()`:
  - Uses **generators** + `yield` with one-row-at-a-time, using a single query with a cursor (or chunk under the hood).
  - Very memory efficient, but:
    - No random access.
    - Might hold DB connection longer.

- `lazy()`:
  - Similar to `cursor()` but uses “lazy collections” with chunking.
  - Good compromise between memory and performance.

- `lazyById()`:
  - Uses primary key comparisons instead of offset.
  - More stable when rows are inserted/deleted during iteration.

Senior-level answer: you know trade-offs and when to pick **lazyById** over **chunk** for large, changing tables.

---

## 3. Performance & Optimization

### Q11. How do you improve Eloquent performance in a high-traffic app?

**Answer:**

Key techniques:

1. **Select only needed columns**:
   ```php
   User::select('id', 'name')->get();
   ```

2. **Eager load relationships** (`with`) and avoid N+1.

3. Use **indexes** at DB level on columns used in `WHERE`, `JOIN`, `ORDER BY`.

4. Use **caching**:
   - `Cache::remember()` for expensive queries.
   - Cache result sets or computed aggregates.

5. Avoid doing heavy work inside **accessors** that run per row (like additional queries).

6. Use **`chunk`/`lazy`** when processing large tables.

7. Use **Query Builder** or raw SQL for super-heavy analytics instead of building huge Eloquent graphs.

8. Avoid `->toArray()` or serializing thousands of full models unnecessarily; use pagination and projection.

---

### Q12. When would you choose Query Builder over Eloquent for performance reasons?

**Answer:**

Use **Query Builder** when:

- You don’t need model features (events, accessors, relationships).
- You’re doing **heavy aggregations**:
  - complex `GROUP BY`, `HAVING`, window functions
- You’re returning data for **reports or exports** where strong typing / model behavior doesn’t matter.
- You want to avoid overhead of instantiating thousands of model objects.

Example:

```php
DB::table('orders')
  ->selectRaw('status, COUNT(*) as total, SUM(amount) as amount_sum')
  ->groupBy('status')
  ->get();
```

This is more efficient and clearer than trying to do the same via Eloquent models.

---

### Q13. How do you handle “fat models” and keep Eloquent code maintainable?

**Answer:**

- Use **Service classes / Actions** for complex workflows, not controllers or models.
- Keep models focused on:
  - ORM mapping
  - Relationships
  - Simple domain helpers
  - Scopes

Example refactor:

```php
// Instead of putting this huge method on the User model:
User::registerWithWelcomeEmail($data);

// Create a dedicated action/service:
class RegisterUserAction
{
    public function execute(array $data): User
    {
        return DB::transaction(function () use ($data) {
            $user = User::create([...]);
            // other stuff...
            return $user;
        });
    }
}
```

Senior-level: demonstrate awareness of **separation of concerns** and how to avoid a “god model”.

---

## 4. Soft Deletes, Timestamps & Model Events

### Q14. How do soft deletes work in Eloquent, and what are the gotchas?

**Answer:**

- Add `use SoftDeletes;` trait on the model.
- Requires `deleted_at` nullable timestamp column.
- Soft-deleted rows are **excluded by default**.

Usage:

```php
Post::all();             // excludes soft-deleted
Post::withTrashed();     // includes soft-deleted
Post::onlyTrashed();     // only soft-deleted
$post->restore();        // restore soft-deleted
$post->forceDelete();    // permanently delete
```

Gotchas:

- Unique constraints may still fail if soft-deleted rows exist.
- `withTrashed()` is often needed in relationships (e.g. to show who deleted something).
- Always consider how soft deletes impact reports and foreign keys.

---

### Q15. What are model events and observers, and when would you use them?

**Answer:**

**Model events**: `creating`, `created`, `updating`, `updated`, `deleting`, `deleted`, `restoring`, `restored`, etc.

```php
protected static function booted()
{
    static::creating(function ($model) {
        // set defaults, audit, validation
    });
}
```

**Observers**: group event handlers into a separate class.

```php
class UserObserver
{
    public function created(User $user)
    {
        // send welcome email, etc.
    }
}

// AppServiceProvider
User::observe(UserObserver::class);
```

Use them for:

- Auditing / logging.
- Enforcing invariants (e.g. always set `tenant_id`).
- Side effects tied to single model changes.

Senior-level: Don’t put heavy business workflows here (better in service layer), but use them for **data consistency** and small hooks.

---

## 5. Bulk Operations, Upserts & Concurrency

### Q16. What is `upsert` in Eloquent, and when would you use it?

**Answer:**

`upsert()` performs **insert or update** in bulk based on unique keys.

```php
User::upsert(
    [
        ['email' => 'a@example.com', 'name' => 'A'],
        ['email' => 'b@example.com', 'name' => 'B'],
    ],
    ['email'],            // unique key(s)
    ['name']              // columns to update if exists
);
```

Use it when:

- Importing external data.
- Syncing lists.
- Avoiding multiple `insert` + `updateOrCreate` loops.

Senior concern: `upsert` bypasses model events/accessors, so be careful with logic that relies on them.

---

### Q17. How do you handle concurrency and race conditions with Eloquent?

**Answer:**

Patterns:

1. **DB transactions**:

   ```php
   DB::transaction(function () use ($data) {
       $order = Order::lockForUpdate()->find($id);
       // modify...
       $order->save();
   });
   ```

2. **Pessimistic locking** (`lockForUpdate`, `sharedLock`) to prevent concurrent writes on the same row.

3. **Optimistic locking**:
   - Add a `version` column.
   - Check that `version` hasn’t changed before update.
   - Increment version on each update.

4. **Atomic DB operations**:
   - `increment`, `decrement`, `update` with where conditions.

   ```php
   User::where('id', $id)->where('balance', $oldBalance)
       ->update(['balance' => $newBalance]);
   ```

Senior-level: be able to explain **why race conditions happen** and how to detect & mitigate them.

---

## 6. Polymorphic & Advanced Modeling

### Q18. When would you use polymorphic relationships, and what are the pros/cons?

**Answer:**

Example: `comments` belonging to multiple models (`posts`, `videos`, `products`):

```php
class Comment extends Model
{
    public function commentable()
    {
        return $this->morphTo();
    }
}
```

Pros:

- One `comments` table for many commentable types.
- Easy to add more commentable models.

Cons:

- Harder to enforce referential integrity (no foreign key constraint on `commentable_id`).
- Harder to do cross-type queries and indexing.
- Can get messy with too many types.

Senior-level: choose polymorphic when **flexibility** outweighs strict FK; otherwise use explicit pivot tables or separate tables.

---

### Q19. How can you model value objects or complex types with Eloquent?

**Answer:**

Use **custom casts**:

```php
class MoneyCast implements CastsAttributes
{
    public function get($model, string $key, $value, array $attributes)
    {
        return new Money($value); // your value object
    }

    public function set($model, string $key, $value, array $attributes)
    {
        return [$key => $value->toInt()];
    }
}

class Order extends Model
{
    protected $casts = [
        'amount' => MoneyCast::class,
    ];
}
```

Benefits:

- Stronger typing in your domain layer.
- Encapsulated logic (e.g. currency, rounding) in value objects.

---

## 7. Testing & Best Practices

### Q20. How do you test Eloquent models and queries effectively?

**Answer:**

- Use **factories**:

  ```php
  $user = User::factory()->create();
  $orders = Order::factory()->count(3)->for($user)->create();
  ```

- Use `RefreshDatabase` or `DatabaseTransactions` traits.
- Test:
  - Relationships (`$user->orders` returns expected count).
  - Scopes (`User::active()`).
  - Constraints (unique keys, foreign keys).
  - Business rules in services that use Eloquent.

Senior-level: keep Eloquent tests **fast** and isolate heavy integration tests; don’t rely on production DB.

---

### Q21. When would you *not* use Eloquent at all?

**Answer:**

- Very heavy reporting / analytics:
  - Better to use raw SQL, views, or a dedicated reporting DB.
- Complex multi-table joins with custom select/projections where Eloquent brings little value.
- Performance-sensitive pipelines where ORM overhead is measurable and unnecessary.

In those cases:

```php
DB::select('SELECT ...'); // or DB::table()->...
```

Senior point: Eloquent is great for domain modeling, not always the best for **every** query.

---

### Q22. How would you implement multi-tenancy with Eloquent?

**Answer (one approach):**

- Add `tenant_id` to all tenant-scoped tables.
- Use a **global scope**:

  ```php
  class TenantScope implements Scope
  {
      public function apply(Builder $builder, Model $model)
      {
          if ($tenantId = tenant()->id ?? null) {
              $builder->where($model->getTable().'.tenant_id', $tenantId);
          }
      }
  }
  ```

- In each tenant-aware model:

  ```php
  protected static function booted()
  {
      static::addGlobalScope(new TenantScope);
  }

  protected static function boot()
  {
      parent::boot();

      static::creating(function ($model) {
          $model->tenant_id = tenant()->id;
      });
  }
  ```

- Use `withoutGlobalScope()` for admin/superuser queries.

Senior-level: also mention other strategies (separate DB per tenant, schema per tenant) and trade-offs.

---

You can save this content as `laravel-eloquent-senior-interview.md` and use it as a study guide or share it with others.
