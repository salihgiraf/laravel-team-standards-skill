# Laravel Team Coding Guidelines

**Version 1.0** — Living document. If you want to change something here, raise a PR.

This document is for our team. If a new developer opens any module in any of our projects, they should be able to figure out where things are, how the data moves, and what "finished work" means. We follow the same rules everywhere. Personal style comes second.

---

## 1. Architecture: How a Request Moves Through the App

Every request goes through these layers in this order. **Do not skip a layer.**

```
Route → FormRequest → Controller → Service → Repository → Model
                          ↓
                      Resource (formats the response)
```

| Layer | What it does | What it must NOT do |
|---|---|---|
| **Route** | Connects a URL to a controller method | Hold logic inside a closure |
| **FormRequest** | Checks the input and user permission | Hold business logic |
| **Controller** | Takes the request, calls the service, returns the response | Run queries, hold logic, validate inline |
| **Service** | Business rules, transactions, events, jobs | Touch the `Request` object or write queries |
| **Repository** | All Eloquent and DB queries, with eager loading | Decide business rules |
| **Model** | Relationships, casts, scopes, accessors | Hold feature queries |
| **Resource** | Shapes every JSON or Inertia response | Fire queries (use `whenLoaded()`) |

### Controller rules

Keep controllers small. Usually 3 to 5 lines per method:

```php
public function store(StoreProductRequest $request): JsonResponse
{
    $product = $this->service->create($request->validated());

    return $this->success(new ProductResource($product), 'Product created', 201);
}
```

Controllers must never:

- Call `Model::where()`, `Model::query()`, or `DB::` directly
- Use `$request->validate([...])` inline — always use a FormRequest class
- Return a raw model or array — always use a Resource
- Have `if-else` logic beyond simple flow control (auth check, redirect)

### Service rules

- Inject the **repository interface** in the constructor. Never the concrete class.
- Accept a plain `array` (from `$request->validated()`) or a DTO. Never the `Request` object itself.
- Wrap multi-step writes in `DB::transaction(fn () => ...)`.
- Fire events and dispatch jobs from here, not from controllers.
- Return Models or Collections. The controller wraps them in a Resource.

```php
class ProductService
{
    public function __construct(
        private readonly ProductRepositoryInterface $repository
    ) {}

    public function create(array $data): Product
    {
        return DB::transaction(function () use ($data) {
            $product = $this->repository->create($data);
            ProductCreated::dispatch($product);

            return $product;
        });
    }
}
```

### Repository rules

- Extend `BaseRepository`. Implement a module-specific interface.
- **All eager loading stays in the repository.** Right next to the query.
- Do not leak query builders out. Return a Model, a Collection, or a Paginator.
- Do not write `bind()` calls by hand for each repository — use auto-binding (see §1.1).

### 1.1 Repository Auto-Binding (team standard)

Writing a manual `$this->app->bind(...)` for every new repository is noise. Somebody always forgets. So we bind them automatically.

**Naming rule (must follow):**

| Type | Namespace | Suffix |
|---|---|---|
| Interface | `App\Repositories\Interfaces` | `RepositoryInterface` |
| Implementation | `App\Repositories\Eloquent` | `Repository` |

Example: `App\Repositories\Interfaces\ProductRepositoryInterface` pairs with `App\Repositories\Eloquent\ProductRepository`.

**Code in `app/Providers/AppServiceProvider.php`:**

```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->bindRepositories();
    }

    /**
     * Bind every {Name}RepositoryInterface to its matching {Name}Repository.
     * Runs once on boot. No manual registration needed.
     */
    protected function bindRepositories(): void
    {
        $interfacesPath = app_path('Repositories/Interfaces');

        if (! is_dir($interfacesPath)) {
            return;
        }

        foreach (glob($interfacesPath . '/*RepositoryInterface.php') as $file) {
            $name = basename($file, 'Interface.php'); // ProductRepository

            $interface      = "App\\Repositories\\Interfaces\\{$name}Interface";
            $implementation = "App\\Repositories\\Eloquent\\{$name}";

            if (interface_exists($interface) && class_exists($implementation)) {
                $this->app->bind($interface, $implementation);
            }
        }
    }
}
```

**Result:** make a new `ProductRepositoryInterface` plus a `ProductRepository` in the right folders. That is all the wiring you need. The container resolves the interface when the service asks for it.

**When NOT to use auto-binding:**

- The interface has more than one implementation (for example, an SMS gateway interface pointing to different providers per environment). Bind it by hand.
- The implementation needs constructor arguments beyond the Model. Use a bind closure.

In those cases, add an explicit `bind()` call below the loop and leave a one-line comment on why.

### N+1 prevention (team rule)

If you add a relation field to a Resource, you **must** update the repository's eager loads in the **same commit**. Resources always use `whenLoaded()`, so a missing eager load shows up as missing data in the review — not as a silent N+1 that nobody notices for weeks.

---

## 2. Naming Conventions

| Thing | Rule | Example |
|---|---|---|
| Controllers | Singular + `Controller` | `ProductController` |
| Models | Singular PascalCase | `OrderItem` |
| Tables | Plural snake_case | `order_items` |
| Pivot tables | Singular, alphabetical | `product_tag` |
| Columns | snake_case, no model prefix | `created_at`, not `product_created_at` |
| Foreign keys | `{model}_id` | `user_id` |
| FormRequests | Action + Model + `Request` | `StoreProductRequest`, `UpdateProductRequest` |
| Resources | Model + `Resource` | `ProductResource` |
| Services | Model + `Service` | `ProductService` |
| Repositories | Model + `Repository` / `RepositoryInterface` | `ProductRepository` |
| Routes (URI) | Plural kebab-case | `/order-items` |
| Route names | Dot notation | `products.index`, `products.show` |
| Methods | camelCase, verb first | `getActiveProducts()` |
| Booleans | `is / has / can` prefix | `is_published`, `hasStock()` |
| Events | Past tense | `ProductCreated` |
| Jobs | Imperative | `SendInvoiceEmail` |
| Config keys | snake_case | `services.razorpay.key` |

**Migrations:** never edit a migration that has been merged to `develop`. Make a new migration instead.

---

## 3. Code Style and Static Analysis

- **PSR-12 is a must.** Run **Laravel Pint** (`./vendor/bin/pint`) before every commit. Add it to your git pre-commit hook or let CI handle it.
- **Use PHP 8.3+ features:** constructor property promotion, `readonly`, enums, match expressions, first-class callable syntax.
- **Type everything.** Parameters, return types, properties. All new files start with `declare(strict_types=1);`.
- **Larastan (PHPStan) level 6 or above** must pass in CI. If you add a `@phpstan-ignore`, leave a comment saying why.
- No commented-out code in merged PRs. Git already remembers old code.
- No `dd()`, `dump()`, `ray()`, or `\Log::info('here')` left behind in merged code.

### Env and config

- Do not call `env()` outside of `config/*` files. Feature code reads `config('services.x.key')`.
- Every new `.env` key must be added to `.env.example` in the same PR.
- Secrets never go in git. Not in comments, not in seeders, not in test fixtures.

---

## 4. Database and Eloquent

- **Migrations are the source of truth for the schema.** Write a `down()` method that actually reverses the change.
- Always add indexes for foreign keys, and for columns used in `WHERE` or `ORDER BY` on big tables.
- Use `$fillable` on all models. Do not use `$guarded = []`.
- Keep casts in the model's `casts()` method — dates, enums, booleans, JSON.
- Use query scopes on the model for reusable *conditions*. Use repositories for full *queries*.
- Chunk big operations: `chunkById()` for updates, `lazyById()` for reads, queued jobs for anything slow.
- Use soft deletes only when the business actually needs recovery or history. Not as a default.

---

## 5. API Standards

- Version all mobile and external APIs: `routes/api/v1/{feature}.php`, mounted under `Route::prefix('v1')`.
- Use **Sanctum** for mobile auth.
- Every response goes through the `ApiResponse` trait — `success()`, `error()`, `paginated()` — so the envelope is always the same:

```json
{
  "success": true,
  "message": "Product created",
  "data": { }
}
```

- Use correct HTTP status codes: `201` for create, `204` for delete with no body, `422` for validation, `403` for authorization, `404` for not found.
- Paginate every list endpoint. No unbounded `->get()` on user-facing lists.
- Filter and sort inputs are validated like any other input — through a FormRequest.

### HTTP verbs — when to use what

| Verb | Purpose | Has body | Idempotent |
|---|---|---|---|
| `GET` | Read | No | Yes |
| `POST` | Create, or any non-CRUD action | Yes | No |
| `PUT` | Replace the full resource | Full resource | Yes |
| `PATCH` | Update part of the resource | Only changed fields | Yes |
| `DELETE` | Remove | No | Yes |

**We use `PATCH` for updates by default.** Edit forms usually send only the fields the user changed, so `PATCH` fits better. Keep `PUT` for the rare case where the client really replaces the whole record (for example, syncing a full settings object).

**Do NOT use `PUT` or `PATCH` for non-CRUD actions.** Verbs like "publish", "approve", "cancel", "restore", or "assign" are not updates. They are state changes or commands. Use `POST` on a sub-resource URL:

```
POST   /api/v1/orders/{order}/cancel      ✅  correct
PATCH  /api/v1/orders/{order}             ❌  hides the real intent
POST   /api/v1/posts/{post}/publish       ✅  correct
PATCH  /api/v1/posts/{post}               ❌  wrong — this is a state change, not an edit
```

Simple rule: **if the action has a verb in the business domain, it is a `POST` to a named sub-route.** If it is just "the user edited some fields on a form", it is `PATCH`.

### Route paths — naming and structure

Follow REST. Every resource route file looks the same, so anyone on the team can guess the URLs without reading them.

**Resource conventions:**

| Method | URI | Route name | Controller action |
|---|---|---|---|
| `GET` | `/products` | `products.index` | `index()` |
| `GET` | `/products/create` | `products.create` | `create()` |
| `POST` | `/products` | `products.store` | `store()` |
| `GET` | `/products/{product}` | `products.show` | `show()` |
| `GET` | `/products/{product}/edit` | `products.edit` | `edit()` |
| `PATCH` | `/products/{product}` | `products.update` | `update()` |
| `DELETE` | `/products/{product}` | `products.destroy` | `destroy()` |

**Rules:**

- Plural, kebab-case URIs: `/order-items`, not `/orderItem` or `/order_items`.
- Route parameters are singular and match the model name: `{product}`, not `{id}`. This enables route-model binding.
- Route names use dot notation: `products.update`, `admin.products.index`.
- Nest at most one level. `/products/{product}/reviews` is fine. `/products/{product}/reviews/{review}/comments` is too deep — flatten it to `/reviews/{review}/comments`.
- Custom actions are `POST` on named sub-routes: `/products/{product}/duplicate`, `/orders/{order}/refund`.

**Full example — in `routes/web.php` or a split file:**

```php
Route::middleware('auth')->group(function () {
    // Standard RESTful resource
    Route::resource('products', ProductController::class);

    // Custom domain actions (POST, named sub-routes)
    Route::post('products/{product}/duplicate', [ProductController::class, 'duplicate'])
        ->name('products.duplicate');

    Route::post('products/{product}/publish', [ProductController::class, 'publish'])
        ->name('products.publish');

    // Nested (one level only)
    Route::resource('products.reviews', ProductReviewController::class)
        ->only(['index', 'store']);
});
```

### Splitting route files by URL prefix

One giant `web.php` becomes unreadable fast. We split route files by top-level URL prefix (and by role or guard for admin areas). The top-level `web.php` and `api.php` only load the pieces.

**Folder layout:**

```
routes/
├── web.php                     ← thin: loads the sub-files below
├── api.php                     ← thin: mounts v1 under prefix('v1')
├── web/
│   ├── dashboard.php           ← /dashboard/*
│   ├── products.php            ← /products/*
│   ├── orders.php              ← /orders/*
│   ├── admin.php               ← /admin/*  (own middleware and prefix)
│   └── profile.php             ← /profile/*
└── api/
    └── v1/
        ├── auth.php            ← /api/v1/auth/*
        ├── products.php        ← /api/v1/products/*
        ├── orders.php          ← /api/v1/orders/*
        └── jewellers.php       ← /api/v1/jewellers/*
```

**`routes/web.php` — the thin loader:**

```php
use Illuminate\Support\Facades\Route;

Route::get('/', fn () => redirect()->route('dashboard.index'));

Route::middleware(['auth', 'verified'])->group(function () {
    require __DIR__ . '/web/dashboard.php';
    require __DIR__ . '/web/products.php';
    require __DIR__ . '/web/orders.php';
    require __DIR__ . '/web/profile.php';
});

Route::middleware(['auth', 'verified', 'can:access-admin'])
    ->prefix('admin')
    ->as('admin.')
    ->group(base_path('routes/web/admin.php'));
```

**`routes/web/products.php` — feature-scoped file:**

```php
use App\Http\Controllers\ProductController;
use Illuminate\Support\Facades\Route;

Route::prefix('products')->as('products.')->group(function () {
    Route::get('/', [ProductController::class, 'index'])->name('index');
    Route::get('/create', [ProductController::class, 'create'])->name('create');
    Route::post('/', [ProductController::class, 'store'])->name('store');
    Route::get('/{product}', [ProductController::class, 'show'])->name('show');
    Route::get('/{product}/edit', [ProductController::class, 'edit'])->name('edit');
    Route::patch('/{product}', [ProductController::class, 'update'])->name('update');
    Route::delete('/{product}', [ProductController::class, 'destroy'])->name('destroy');

    Route::post('/{product}/duplicate', [ProductController::class, 'duplicate'])->name('duplicate');
});
```

**`routes/api.php` — mounts v1:**

```php
use Illuminate\Support\Facades\Route;

Route::prefix('v1')->as('api.v1.')->group(function () {
    require __DIR__ . '/api/v1/auth.php';

    Route::middleware('auth:sanctum')->group(function () {
        require __DIR__ . '/api/v1/products.php';
        require __DIR__ . '/api/v1/orders.php';
        require __DIR__ . '/api/v1/jewellers.php';
    });
});
```

**Rules for splitting:**

- One file per top-level URL prefix. If a feature adds a new top-level path like `/inventory`, create `routes/web/inventory.php`.
- Apply the prefix, name-prefix, and middleware **at the point of `require`**. Do not repeat them inside the file. This keeps each file focused on the routes themselves.
- Admin, tenant, and guest areas each get their own file with their own guard middleware.
- Do not split too much. One route per file is overkill. Group by feature, not by controller action.

---

## 6. Exception Handling (in `bootstrap/app.php`)

Laravel 11 and above moved exception handling out of `app/Exceptions/Handler.php` and into `bootstrap/app.php` via `->withExceptions()`. This is the one place where all custom exception handling lives. Do **not** bring back a `Handler.php` file. Do **not** use try-catch in controllers or services just to change the response shape.

**Principles:**

- Business and domain code `throw`s specific exceptions (`ProductNotAvailableException`, `PaymentFailedException`, and so on). It does not worry about how to format the response.
- `bootstrap/app.php` maps every exception to the right JSON envelope (for API) or Inertia response (for web).
- API errors use the same `success` / `error` envelope as normal responses. A mobile client can parse every response the same way.

**`bootstrap/app.php`:**

```php
<?php

use App\Exceptions\BusinessException;
use Illuminate\Auth\AuthenticationException;
use Illuminate\Auth\Access\AuthorizationException;
use Illuminate\Database\Eloquent\ModelNotFoundException;
use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;
use Illuminate\Http\Request;
use Illuminate\Validation\ValidationException;
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;
use Symfony\Component\HttpKernel\Exception\ThrottleRequestsException;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web:      __DIR__ . '/../routes/web.php',
        api:      __DIR__ . '/../routes/api.php',
        commands: __DIR__ . '/../routes/console.php',
        health:   '/up',
    )
    ->withMiddleware(function (Middleware $middleware) {
        // Global middleware config here
    })
    ->withExceptions(function (Exceptions $exceptions) {
        // Helper — is this an API request?
        $isApi = fn (Request $request) => $request->is('api/*') || $request->expectsJson();

        // 401 — not authenticated
        $exceptions->render(function (AuthenticationException $e, Request $request) use ($isApi) {
            if ($isApi($request)) {
                return response()->json([
                    'success' => false,
                    'message' => 'Unauthenticated.',
                ], 401);
            }
        });

        // 403 — authenticated but not allowed
        $exceptions->render(function (AuthorizationException $e, Request $request) use ($isApi) {
            if ($isApi($request)) {
                return response()->json([
                    'success' => false,
                    'message' => $e->getMessage() ?: 'This action is unauthorized.',
                ], 403);
            }
        });

        // 404 — model or route not found
        $exceptions->render(function (ModelNotFoundException $e, Request $request) use ($isApi) {
            if ($isApi($request)) {
                return response()->json([
                    'success' => false,
                    'message' => 'Resource not found.',
                ], 404);
            }
        });

        $exceptions->render(function (NotFoundHttpException $e, Request $request) use ($isApi) {
            if ($isApi($request)) {
                return response()->json([
                    'success' => false,
                    'message' => 'Endpoint not found.',
                ], 404);
            }
        });

        // 422 — validation
        $exceptions->render(function (ValidationException $e, Request $request) use ($isApi) {
            if ($isApi($request)) {
                return response()->json([
                    'success' => false,
                    'message' => 'The given data was invalid.',
                    'errors'  => $e->errors(),
                ], 422);
            }
        });

        // 429 — rate limit
        $exceptions->render(function (ThrottleRequestsException $e, Request $request) use ($isApi) {
            if ($isApi($request)) {
                return response()->json([
                    'success' => false,
                    'message' => 'Too many requests. Please try again shortly.',
                ], 429);
            }
        });

        // Domain exceptions — every BusinessException carries its own code and message
        $exceptions->render(function (BusinessException $e, Request $request) use ($isApi) {
            if ($isApi($request)) {
                return response()->json([
                    'success' => false,
                    'message' => $e->getMessage(),
                ], $e->getCode() ?: 400);
            }

            return back()->withErrors(['error' => $e->getMessage()])->withInput();
        });

        // Fallback — anything else in production becomes a generic 500 for API clients
        $exceptions->render(function (\Throwable $e, Request $request) use ($isApi) {
            if ($isApi($request) && ! config('app.debug')) {
                return response()->json([
                    'success' => false,
                    'message' => 'Something went wrong. Please try again later.',
                ], 500);
            }
        });

        // Do not log noisy, expected framework exceptions
        $exceptions->dontReport([
            ValidationException::class,
            AuthenticationException::class,
            AuthorizationException::class,
            ModelNotFoundException::class,
            NotFoundHttpException::class,
            ThrottleRequestsException::class,
        ]);
    })
    ->create();
```

**Custom `BusinessException` base class:**

Keep domain exceptions in `app/Exceptions/` and extend a shared base:

```php
namespace App\Exceptions;

use Exception;

class BusinessException extends Exception
{
    public function __construct(string $message = '', int $code = 400)
    {
        parent::__construct($message, $code);
    }
}

// A concrete one:
class ProductOutOfStockException extends BusinessException
{
    public function __construct(string $sku)
    {
        parent::__construct("Product {$sku} is out of stock.", 409);
    }
}
```

**Usage in a service:**

```php
if ($product->stock < $quantity) {
    throw new ProductOutOfStockException($product->sku);
}
```

The controller does not `try/catch` this. `bootstrap/app.php` turns it into a clean `409` response with the right envelope on its own.

**Rules:**

- Services and repositories `throw`. Controllers do not `try/catch` domain exceptions.
- Every new domain exception extends `BusinessException` (or a more specific base). Never `throw new \Exception('...')` in feature code.
- The `render()` callbacks in `bootstrap/app.php` only decide the **response shape**. They do not hold business logic.
- Anything unexpected (uncaught `Throwable`) returns a generic 500 in production. Details go to the logs. Never show them to the user.

---

## 7. Frontend (Inertia + React + TypeScript + Tailwind v4)

- **TypeScript everywhere.** Keep shared types in `resources/js/types/index.ts`, typed from the Resource shapes.
- Pages live in `resources/js/Pages/{Module}/Index.tsx | Create.tsx | Edit.tsx | Show.tsx`.
- Use `useForm`, `Link`, and `router` from `@inertiajs/react`. Do not hand-roll `fetch` for Inertia navigation.
- Tailwind v4 is CSS-first. Keep tokens in `resources/css/app.css` under a `@theme { }` block. No `tailwind.config.js` unless a plugin forces it.
- Extract a component when the same utility string appears 3 or more times. Use `cn()` or `clsx` for conditional classes.
- Reusable primitives go in `Components/ui/`. Feature components stay near their pages.

---

## 8. Git Workflow

### Branches

```
main        → production (deploy only from here)
develop     → integration branch
feature/*   → feature/gold-rate-poster
fix/*       → fix/invoice-total-rounding
hotfix/*    → hotfix/payment-webhook-500 (branches from main)
```

- Branch names: lowercase kebab-case, with a type prefix. Include the ticket ID if you use a tracker: `feature/YM-142-service-report-pdf`.
- Keep branches short. Merge within a few days. Rebase on `develop` before opening a PR.
- Never commit directly to `main` or `develop`.

### Commits — Conventional Commits

```
feat: add per-jeweller gold rate poster caching
fix: prevent duplicate service invoice on retry
refactor: extract feedback saving into FeedbackService
chore: bump pint to 1.18
docs: update module scaffolding guide
test: add coverage for promo code expiry
```

- Present tense, imperative mood, under 72 characters for the subject.
- One logical change per commit. "fix stuff" and "wip" do not get merged.
- The body (optional) explains *why*, not *what*. The diff already shows what.

---

## 9. Pull Requests and Code Review

### PR checklist (author)

Before you ask for review, check:

- [ ] Pint passes, Larastan passes, tests pass on your machine
- [ ] The layer flow is respected (no queries in controllers or services)
- [ ] New Resource relations have matching eager loads in the repository
- [ ] FormRequest `authorize()` does a real policy check — not `return true`
- [ ] `.env.example` is updated if you added new env keys
- [ ] Migrations run cleanly up **and** down
- [ ] Screenshots or a recording are attached for any UI change
- [ ] The PR description says *what* changed and *why*

### Review rules (reviewer)

- At least **one approval** is needed to merge. Two approvals for anything touching payments, auth, or migrations on production data.
- Review within 24 hours, or say you cannot. Do not leave PRs hanging.
- Comment on the code, not the person. Prefer questions: "What happens if this array is empty?" instead of "This is wrong."
- Mark nitpicks clearly (`nit:`). Nits should not block a merge.
- The author merges after approval. The reviewer never merges someone else's PR.
- Keep PRs under 400 changed lines when you can. Split big features into stacked PRs.

### Anti-patterns that block a PR

- `Model::where(...)` inside a controller or a service
- Inline `$request->validate([...])`
- Raw models or arrays returned from a controller
- Services type-hinting concrete repositories instead of interfaces
- Eager loads placed in the controller
- Business logic inside a FormRequest or a Resource
- `env()` calls outside config files

---

## 10. Testing

- **Every feature PR comes with tests.** Minimum bar: feature tests for each new endpoint (happy path, validation failure, authorization failure).
- Use **Pest** (or PHPUnit if the project is older) with `RefreshDatabase`.
- Use factories for all models. Tests never insert rows by hand.
- Unit-test services with mocked repository interfaces. Feature-test the full HTTP flow.
- Name tests by behaviour: `it_prevents_duplicate_invoice_numbers()`, not `test_invoice_2()`.
- Do not test the framework (for example, that `required` rules reject empty strings — one endpoint test covers that; no need for exhaustive rule-by-rule tests).
- CI runs Pint, Larastan, and tests on every PR. Red CI = no merge. No exceptions.

---

## 11. Security Essentials

- Authorization through **Policies**, called from FormRequest `authorize()` or from a controller `authorize()` call. Never trust route or model IDs without a policy check.
- Mass assignment protection through `$fillable` + `$request->validated()`. Never pass `$request->all()` into `create()`.
- All file uploads: validate mime type and size. Store with the `Storage` facade using a generated name. Never keep the original filename.
- Queue anything that sends email or SMS, or calls an external API. Configure retry and backoff.
- Rate-limit auth and public API endpoints with the `throttle` middleware.
- User-visible errors are generic. Details go to the logs. `APP_DEBUG=false` in production. Always.

---

## 12. Performance Defaults

- Eager load everything a Resource reads (see §1).
- Cache expensive derived data with clear keys and TTLs. Invalidate the cache in the service that writes the source data.
- Long-running work (PDF or poster generation, imports, notifications) always goes through a **queued job**. Never run it inline in the request.
- Use `select()` to limit columns on wide tables in list queries.
- Add `laravel-debugbar` locally. If a page fires more than 15 queries, treat it as a bug.

---

## 13. Scaffolding a New Module

When you make a new module (say `Product`), create all of these together:

1. Migration and Model (`$fillable`, relationships, casts)
2. `ProductRepositoryInterface` (extends `EloquentRepositoryInterface`)
3. `ProductRepository` (extends `BaseRepository`, all eager loading here)
4. `ProductService` (injects the interface, handles transactions and events)
5. `StoreProductRequest` and `UpdateProductRequest`
6. `ProductResource` (with `whenLoaded()`)
7. Controller — web (Inertia) and/or API (`ApiResponse` trait)
8. Binding — **skip if names follow the convention** (`AppServiceProvider` auto-binds; see §1.1). Add an explicit bind only when the interface has multiple implementations.
9. Routes — new file under `routes/web/products.php` and/or `routes/api/v1/products.php`. Require it from `web.php` or `api.php`.
10. Inertia pages (`Pages/Product/*.tsx`) with typed props.

A module PR missing any of these is incomplete.

---

## 14. Documentation and Communication

- Every module gets a short `README` note (or a section in the project wiki): what it does, key business rules, anything non-obvious.
- Announce breaking changes (schema, API contract, env keys) in the team channel **before** merging.
- If you are stuck for more than 30 minutes, ask. If you find a gotcha, write it down where the next person will find it.
- Update this document when the team agrees on a new rule. Guidelines that live only in people's heads do not count.
