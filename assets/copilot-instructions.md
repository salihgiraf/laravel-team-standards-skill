# Project Coding Standards for GitHub Copilot

This is a Laravel 11+ project with Inertia.js, React, TypeScript, and Tailwind CSS v4. When suggesting code, follow these rules.

## Architecture

- Request flow: `Route → FormRequest → Controller → Service → Repository → Model`, with `Resource` formatting the response.
- Never skip a layer.

## Controllers

- Keep methods under 5 lines where possible.
- Never use `Model::query()`, `Model::where()`, or `DB::` directly in a controller.
- Never use `$request->validate([...])` inline. Always use a FormRequest class.
- Always wrap the return value in a Resource.
- Inject a Service in the constructor; delegate work to it.

## Services

- Inject the **repository interface**, never the concrete class.
- Accept a plain array from `$request->validated()`, never the `Request` object.
- Wrap multi-step writes in `DB::transaction(fn () => ...)`.
- Fire events and dispatch jobs here, not in controllers.
- Return Models or Collections; let the controller wrap them in a Resource.

## Repositories

- Extend `BaseRepository`. Implement a module-specific interface.
- All eager loading (`with(...)`) lives in the repository, not in the controller or service.
- Return Models, Collections, or Paginators — never raw query builders.

## Auto-binding naming contract

- Interface: `App\Repositories\Interfaces\{Name}RepositoryInterface`
- Implementation: `App\Repositories\Eloquent\{Name}Repository`

Example: `ProductRepositoryInterface` pairs with `ProductRepository`. `AppServiceProvider` binds them automatically. Do not write manual `$this->app->bind(...)` for new repositories.

## FormRequests

- One request per action: `StoreProductRequest`, `UpdateProductRequest`.
- `authorize()` must do a real policy check, not `return true`.
- Use `prepareForValidation()` for input normalization.

## Resources

- Use `whenLoaded('relation')` for all relations.
- Never trigger a query from inside a Resource.

## Routes

- Use `PATCH` for updates (edit forms). Use `POST` for state changes (`/orders/{order}/cancel`, `/posts/{post}/publish`).
- Plural, kebab-case URIs: `/order-items`, not `/orderItem`.
- Route parameters match model names: `{product}`, not `{id}`.
- Route names use dot notation: `products.update`, `admin.products.index`.
- Split route files by URL prefix under `routes/web/` and `routes/api/v1/`, loaded from a thin `routes/web.php` / `routes/api.php`.

## Exception handling

- Laravel 11+ uses `bootstrap/app.php` with `->withExceptions()`. Do not create `app/Exceptions/Handler.php`.
- Services and repositories `throw`. Controllers do not `try/catch` domain exceptions.
- Every domain exception extends `App\Exceptions\BusinessException`. Never `throw new \Exception(...)` in feature code.
- API errors use a consistent envelope: `{ "success": false, "message": "...", "errors": {} }`.

## Naming conventions

- Controllers: `ProductController` (singular + Controller).
- Models: `OrderItem` (singular PascalCase).
- Tables: `order_items` (plural snake_case).
- Foreign keys: `user_id` (`{model}_id`).
- Booleans: prefix with `is_`, `has_`, or `can_` → `is_published`, `hasStock()`.
- Events: past tense → `ProductCreated`.
- Jobs: imperative → `SendInvoiceEmail`.

## PHP style

- PSR-12, PHP 8.3+.
- `declare(strict_types=1);` at the top of every new file.
- Constructor property promotion, `readonly` properties, enums, match expressions.
- Type every parameter, return, and property.
- Never call `env()` outside `config/*`. Use `config('services.x.key')`.

## Database

- Use `$fillable`, never `$guarded = []`.
- Keep casts in the model's `casts()` method.
- Index foreign keys and any column used in `WHERE` or `ORDER BY` on large tables.
- Chunk big operations with `chunkById()` or `lazyById()`.

## API

- Version APIs under `/api/v1/`.
- Use Sanctum for mobile auth.
- Every response goes through the `ApiResponse` trait (`success()`, `error()`, `paginated()`).
- Paginate every list endpoint.

## Frontend (Inertia + React + Tailwind v4)

- TypeScript for all pages and components.
- Pages: `resources/js/Pages/{Module}/{Index,Create,Edit,Show}.tsx`.
- Use `useForm`, `Link`, `router` from `@inertiajs/react`. Do not hand-roll `fetch` for Inertia navigation.
- Tailwind v4: tokens in `resources/css/app.css` inside `@theme { }`. Do not create `tailwind.config.js` unless a plugin needs it.
- Use `cn()` or `clsx` for conditional classes.
- Extract a component when the same utility string repeats 3+ times.

## Anti-patterns to avoid

- `Model::where(...)` in a controller or service.
- Inline `$request->validate([...])`.
- Returning a raw model or array from a controller.
- Type-hinting a concrete repository instead of its interface.
- Eager loading placed in the controller.
- Business logic inside a FormRequest or Resource.
- `try/catch` in a controller just to change the status code.
- `env()` outside of config files.
