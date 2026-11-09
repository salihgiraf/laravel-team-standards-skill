# Project Coding Standards

This is a Laravel project using Inertia.js + React + Tailwind v4. Every new piece of code must follow the rules below. When something here is unclear or conflicts with a request, flag it rather than guessing.

## Architecture: layer flow

Every request moves through these layers in this order. Do not skip a layer.

```
Route → FormRequest → Controller → Service → Repository → Model
                          ↓
                      Resource (formats the response)
```

| Layer | Does | Must NOT do |
|---|---|---|
| Route | Connects URL to controller | Hold logic inside a closure |
| FormRequest | Validates input, authorizes user | Hold business logic |
| Controller | Takes request, calls service, returns response | Run queries, hold logic, validate inline |
| Service | Business rules, transactions, events, jobs | Touch `Request` object, write queries |
| Repository | All Eloquent / DB queries, eager loading | Decide business rules |
| Model | Relationships, casts, scopes, accessors | Hold feature queries |
| Resource | Shapes the JSON / Inertia response | Fire queries (use `whenLoaded()`) |

## Controller rules

Keep controllers small — usually 3 to 5 lines per method.

```php
public function store(StoreProductRequest $request): JsonResponse
{
    $product = $this->service->create($request->validated());

    return $this->success(new ProductResource($product), 'Product created', 201);
}
```

Controllers must never:
- Call `Model::where()`, `Model::query()`, or `DB::` directly
- Use `$request->validate([...])` inline — always a FormRequest class
- Return a raw model or array — always wrap in a Resource
- Contain `if/else` beyond simple flow control

## Service rules

- Inject the **repository interface** in the constructor, never the concrete class.
- Accept a plain `array` from `$request->validated()` (or a DTO), never the `Request` object.
- Wrap multi-step writes in `DB::transaction(fn () => ...)`.
- Fire events and dispatch jobs from here, not from controllers.
- Return Models or Collections; the controller wraps them.

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

## Repository rules

- Extend `BaseRepository`, implement a module-specific interface.
- **All eager loading lives in the repository**, next to the query.
- Never leak query builders. Return a Model, Collection, or Paginator.
- Do not write `bind()` calls by hand — auto-binding handles it (see below).

## Auto-binding (team standard)

Repository interfaces bind to their implementations automatically in `AppServiceProvider::register()`. Follow the naming contract and no manual registration is needed.

**Naming contract:**

| Type | Namespace | Suffix |
|---|---|---|
| Interface | `App\Repositories\Interfaces` | `RepositoryInterface` |
| Implementation | `App\Repositories\Eloquent` | `Repository` |

`ProductRepositoryInterface` → pairs with `ProductRepository`.

**In `AppServiceProvider`:**

```php
protected function bindRepositories(): void
{
    $interfacesPath = app_path('Repositories/Interfaces');
    if (! is_dir($interfacesPath)) return;

    foreach (glob($interfacesPath . '/*RepositoryInterface.php') as $file) {
        $name = basename($file, 'Interface.php');
        $interface      = "App\\Repositories\\Interfaces\\{$name}Interface";
        $implementation = "App\\Repositories\\Eloquent\\{$name}";

        if (interface_exists($interface) && class_exists($implementation)) {
            $this->app->bind($interface, $implementation);
        }
    }
}
```

Bind by hand only when:
- The interface has multiple implementations (e.g. SMS gateway per environment).
- The implementation needs constructor arguments beyond the Model.

## Routing

### HTTP verbs

| Verb | Use for |
|---|---|
| `GET` | Read |
| `POST` | Create, or any non-CRUD action |
| `PATCH` | Partial update (edit forms — our default for updates) |
| `PUT` | Full-resource replacement (rare) |
| `DELETE` | Remove |

**State changes are `POST`, not `PATCH`.** Verbs like "publish", "cancel", "approve", "refund" are commands, not updates. Use a named sub-route:

```
POST /orders/{order}/cancel      ✅
PATCH /orders/{order}            ❌  hides intent
```

### Route paths

- Plural, kebab-case URIs: `/order-items`, not `/orderItem`.
- Route parameters match the model name: `{product}`, not `{id}` (enables route-model binding).
- Route names use dot notation: `products.update`, `admin.products.index`.
- Nest at most one level.
- Custom actions are `POST` on named sub-routes: `/products/{product}/duplicate`.

### Split route files by URL prefix

```
routes/
├── web.php                     # thin loader
├── api.php                     # thin, mounts v1
├── web/
│   ├── dashboard.php           # /dashboard/*
│   ├── products.php            # /products/*
│   └── admin.php               # /admin/*
└── api/v1/
    ├── auth.php
    ├── products.php
    └── orders.php
```

The thin loader applies prefix, name-prefix, and middleware at the point of `require`. Feature files only hold the routes themselves.

```php
// routes/web.php
Route::middleware(['auth', 'verified'])->group(function () {
    require __DIR__ . '/web/dashboard.php';
    require __DIR__ . '/web/products.php';
});

Route::middleware(['auth', 'can:access-admin'])
    ->prefix('admin')->as('admin.')
    ->group(base_path('routes/web/admin.php'));
```

```php
// routes/web/products.php
Route::prefix('products')->as('products.')->group(function () {
    Route::get('/', [ProductController::class, 'index'])->name('index');
    Route::post('/', [ProductController::class, 'store'])->name('store');
    Route::get('/{product}', [ProductController::class, 'show'])->name('show');
    Route::patch('/{product}', [ProductController::class, 'update'])->name('update');
    Route::delete('/{product}', [ProductController::class, 'destroy'])->name('destroy');
});
```

## Exception handling (in `bootstrap/app.php`)

Laravel 11+ moved exception handling into `bootstrap/app.php` via `->withExceptions()`. Do not bring back `app/Exceptions/Handler.php`. Do not add `try/catch` in controllers or services to reshape responses.

- Services and repositories `throw`. Controllers do not catch.
- Every domain exception extends `App\Exceptions\BusinessException`. Never `throw new \Exception('...')` in feature code.
- `bootstrap/app.php` maps every exception to the right JSON envelope (API) or Inertia response (web).

**Pattern:**

```php
// app/Exceptions/BusinessException.php
class BusinessException extends Exception
{
    public function __construct(string $message = '', int $code = 400)
    {
        parent::__construct($message, $code);
    }
}

// In a service
if ($product->stock < $quantity) {
    throw new ProductOutOfStockException($product->sku);
}
```

The `withExceptions()` block in `bootstrap/app.php` renders all the standard exceptions (401, 403, 404, 422, 429) plus `BusinessException` as the consistent envelope:

```json
{ "success": false, "message": "...", "errors": { } }
```

## Naming conventions

| Thing | Rule | Example |
|---|---|---|
| Controllers | Singular + `Controller` | `ProductController` |
| Models | Singular PascalCase | `OrderItem` |
| Tables | Plural snake_case | `order_items` |
| Columns | snake_case, no model prefix | `created_at` |
| Foreign keys | `{model}_id` | `user_id` |
| FormRequests | Action + Model + `Request` | `StoreProductRequest` |
| Resources | Model + `Resource` | `ProductResource` |
| Services | Model + `Service` | `ProductService` |
| Repositories | Model + `Repository` / `RepositoryInterface` | `ProductRepository` |
| URIs | Plural kebab-case | `/order-items` |
| Route names | Dot notation | `products.index` |
| Methods | camelCase, verb first | `getActiveProducts()` |
| Booleans | `is / has / can` prefix | `is_published` |
| Events | Past tense | `ProductCreated` |
| Jobs | Imperative | `SendInvoiceEmail` |

## API standards

- Version APIs: `routes/api/v1/{feature}.php`, mounted under `Route::prefix('v1')`.
- Use Sanctum for mobile auth.
- Every API response goes through the `ApiResponse` trait (`success()`, `error()`, `paginated()`) for a consistent envelope.
- Paginate every list endpoint. No unbounded `->get()`.

## Frontend (Inertia + React + Tailwind v4)

- TypeScript for all pages and components. Shared types in `resources/js/types/index.ts`.
- Pages: `resources/js/Pages/{Module}/{Index,Create,Edit,Show}.tsx`.
- Use `useForm`, `Link`, `router` from `@inertiajs/react`. No hand-rolled `fetch` for Inertia navigation.
- Tailwind v4 is CSS-first. Tokens in `resources/css/app.css` under `@theme { }`. No `tailwind.config.js` unless a plugin forces it.
- Extract a component when the same utility string appears 3+ times. Use `cn()` / `clsx` for conditional classes.

## N+1 prevention

If you add a relation field to a Resource, update the repository's eager loads in the same commit. Resources use `whenLoaded()`, so a missing eager load shows up as missing data in review — not as a silent N+1.

## Anti-patterns that block a PR

- `Model::where(...)` in a controller or service
- Inline `$request->validate([...])`
- Raw models or arrays returned from a controller
- Service type-hinting the concrete repository instead of the interface
- Eager loading placed in the controller
- Business logic inside a FormRequest or Resource
- `env()` called outside `config/*` files
- `app/Exceptions/Handler.php` re-created (exceptions live in `bootstrap/app.php`)
- `try/catch` in a controller just to change a status code

## Code style

- PSR-12, enforced by Laravel Pint. Run `./vendor/bin/pint` before every commit.
- PHP 8.3+: constructor property promotion, `readonly`, enums, match, first-class callable syntax.
- Type everything. All new files start with `declare(strict_types=1);`.
- Larastan level 6+ must pass in CI.
- No `dd()`, `dump()`, `ray()`, or `\Log::info('here')` in merged code.
- Never call `env()` outside `config/*`. Feature code uses `config('services.x.key')`.
