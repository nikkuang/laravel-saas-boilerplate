# CLAUDE.md — Standing Conventions

> Read this before writing or modifying any code in this repo. It exists so
> every Claude Code session — and every human contributor — produces
> consistent code, regardless of who or what wrote the last commit.

## Architecture, non-negotiable

- **Business logic lives in `app/Services/` or `app/Actions/` — never in
  controllers.** Controllers validate input, call a service, format the
  response. If you find yourself writing `if` statements with business rules
  inside a controller, stop and move it to a service.
- **Input validation lives in Form Request classes (`app/Http/Requests`), not
  inline `$request->validate()` in controllers.** Every endpoint that accepts
  input gets a `FormRequest` — `rules()` for validation, `authorize()` for
  boundary authorization (see Authorization). The controller then converts the
  validated data into a typed `spatie/laravel-data` DTO and passes the **DTO**
  (not `$request->validated()`'s array) into the service — the validated array
  is absorbed at the controller edge and never crosses into the service layer.
  **Validation rules live in exactly one place — the Form Request's
  `rules()`.** The DTO declares **no** `rules()` and does not self-validate;
  it's built from the already-validated data (`SubscriptionData::from($request
  ->validated())`) as a pure typed carrier. No rule is ever written twice.
- **Web (Inertia), API, and Filament may call the same service — that's the
  point.** Don't duplicate logic across them; if a rule changes, it should
  require touching exactly one file. **This includes Filament Resource, Page,
  and Action classes** — business logic goes in the service, not in Filament
  hooks (`mutateFormDataBeforeSave`, action closures, table actions, etc.).
  Filament classes stay as thin as controllers.
- **Never mix Livewire and Inertia on the same page.** Filament owns
  `/admin` (business admin) and `/devtools` (developer/ops panel) — nothing
  outside those uses Livewire. Everything authenticated outside them is
  Inertia + Vue. Everything public/guest is Blade.
- **Never return raw Eloquent models from API controllers.** Always wrap in
  an API Resource.
- **Multi-write operations wrap in `DB::transaction()`, inside the service.**
  Any service method performing more than one write (create-user + issue-token,
  entitlement updates, etc.) is all-or-nothing. Transaction boundaries live in
  the service layer, never spread across a controller.

## Directory layout

Every kind of class has one home — this is what keeps `app/Support` small, by
giving DTOs, enums, and business logic their own place:

| Directory | Holds |
|---|---|
| `app/Services/`, `app/Actions/` | Business logic — orchestration, domain rules, side effects |
| `app/Data/` | `spatie/laravel-data` DTOs (typed layer-boundary payloads) |
| `app/Enums/` | Native PHP enums (`ApiErrorCode`, `AnnouncementType`, …) |
| `app/Support/` | Generic, stateless, domain-agnostic helpers — utility classes, `Str`/`Collection` macros, reusable traits, small value objects |
| `app/Policies/` | Authorization policies (framework default) |
| `app/Filament/{Panel}/` | Filament resources/pages, **one directory per panel** (`Admin/`, `Devtools/`) — never the top-level `app/Filament/Resources` |

- **`app/Support` is not a junk drawer.** If a class knows anything about the
  domain (subscriptions, users, announcements) or carries dependencies, it's a
  Service/Action, not Support. Support holds only what would be equally at home
  in any Laravel project.
- **Each Filament panel owns a directory** under `app/Filament/{Panel}/`
  (e.g. `app/Filament/Admin/Resources`, `app/Filament/Devtools/Resources`),
  with a matching `App\Filament\{Panel}\…` namespace and the panel
  provider's `discoverResources`/`discoverPages` pointed at it. Panels
  **must not share discovery directories** (a shared dir leaks each
  panel's resources into the other). Do **not** leave the default panel at
  the top-level `app/Filament/Resources` — move it under `Admin/` so every
  panel is symmetric. `make:filament-resource` is panel-aware — pass
  `--panel=` (or answer the prompt) so files land in the right tree.
- **Helpers are class-based and typed — no global `helpers.php`.** Use `final`
  classes with static methods (or invokable classes): `App\Support\Money::format(...)`,
  never a global `money_format()`. Global functions are namespace-less,
  un-mockable, and invisible to Larastan at `max`.

## App status & announcements

- Version gating thresholds are **platform-keyed** — never introduce a
  single global `min_supported_version`. Add new values to the `app_settings`
  table + Filament Settings page, not to config files.
- The version-check middleware must **fail open** and skip gracefully when
  version/platform headers are absent (web, Bruno, partner calls). Never let
  it block a request that has no app-version context.
- **Do not add a `maintenance_mode` flag.** Real outages use `php artisan
  down`; planned/advance notices use an announcement. If a task asks for a
  maintenance toggle, use one of those two, and flag the choice.
- Announcement dismissal/seen state is **server-authoritative**, keyed to the
  user via the `announcement_user` pivot — never move the source of truth to
  client-side storage for logged-in users (that reintroduces the
  reinstall/multi-device bug this design exists to prevent). Local storage is
  only a fallback on the pre-login screen.
- Don't add per-user targeting, announcement localization, or read analytics
  beyond `seen_at` without raising it first — these were deliberately scoped
  out of v1.

## API conventions

- All API routes are versioned (`/api/v1/...`). A breaking change gets a new
  version — never break `v1` in place.
- Every API response follows the standard envelope:
  - Success: `{ "data": ..., "meta": ..., "links": ... }`
  - Error: `{ "message": ..., "error_code": ..., "errors": ... }`
- **DTO in, Resource out — one tool per direction.** Inbound: Form Request
  validates → `spatie/laravel-data` DTO carries the typed payload into the
  service. Outbound: the service's result is wrapped in an **API Resource** —
  never return a `Data` DTO directly as the response, even though it can
  serialize itself. Routing every response through a Resource keeps the
  envelope, `whenLoaded` field selection, and versioning path uniform.
- `error_code` values come from the `ApiErrorCode` enum. Adding a new failure
  case means adding a new enum value there first, not inventing a string
  inline.
- Use correct HTTP status codes (see `BOILERPLATE.md` for the full table) —
  don't return 200 with an error in the body.
- Routes are nouns, plural, REST-conventional (`/subscriptions`, not
  `/create-subscription` or `/subscription`). **Exception:** auth and
  state-transition actions (`/login`, `/refresh`, `/app-status`,
  `/announcements/{id}/seen`, `/dismiss`) are intentionally RPC-style verbs —
  the noun rule governs resource CRUD only. Don't "fix" these into contrived
  resources.
- List/index endpoints use `spatie/laravel-query-builder` for
  filtering/sorting/includes. Allow-lists (`allowedFilters`, `allowedSorts`,
  `allowedIncludes`, `allowedFields`) are **mandatory** — deny-by-default,
  never a bare column set. Use it in read controllers only (never pass a
  `QueryBuilder`/`Request` into a service), terminate in `->paginate()`, and
  wrap in a Resource. Annotate the dynamic params so Scramble documents them.
- Any new API endpoint should be covered by Scramble automatically (avoid
  patterns Scramble can't infer — check the generated `openapi.yaml` after
  adding a route to confirm it picked it up correctly). Re-sync the Bruno
  collection after any route change.

## Frontend conventions

- Authenticated pages: Inertia + Vue 3 + Pinia. Follow existing component
  structure in `resources/js/Pages/App/`.
- Marketing/public pages: Blade, in `resources/views/marketing/`. Keep JS
  minimal — Alpine.js only, no Vue bundle on these pages.
- Don't introduce a new state management pattern — Pinia is the only store
  layer. Don't reach for Vuex, don't roll custom reactive state for
  cross-component concerns.
- Tailwind only for styling — no separate CSS framework, no inline `<style>`
  blocks unless truly one-off and component-scoped.
- **Inertia controllers are thin — same shared service layer as the API.**
  Validate (Form Request) → DTO → service → `Inertia::render()`. No business
  logic in the controller; web and mobile hit the same service so rules can't
  drift.
- **The Vue side is TypeScript, typed end to end.** Consume the TS types
  `spatie/laravel-data` generates from your DTOs/Resources, so page props are
  checked against the backend contract at compile time — the frontend half of
  the strict-typing rule.
- **Reuse API Resources as Inertia page props.** The same Resource serializes a
  model for the mobile API and for an Inertia prop — one serialization source,
  identical shapes across clients. Resolve the Resource (`->resolve()` / disable
  `data`-wrapping) so the `JsonResource` `data` key doesn't double-nest as a
  prop. Writes still go Form Request → DTO → service.
- **Global props are shared once via `HandleInertiaRequests::share()`** — auth
  user (minimal, Resource-shaped), flash messages, Pennant flags. Kept lean
  (ships on every response); never refetch per page.
- **Forms use Inertia's `useForm`.** Form Request validation errors flow back
  automatically via Inertia's shared `errors` bag — don't hand-roll `fetch` +
  error handling.
- **UI permission flags come from Policies** — show/hide actions off boolean
  props computed server-side from the same Policies (`$user->can(...)`).
  Vue-side authz is cosmetic; the server still enforces.
- Persistent layouts (no shell re-mount between visits); lazy/partial props
  (`Inertia::lazy()`, `only`) for expensive data. The Inertia app is
  deliberately **not** SSR'd — authenticated dashboards aren't indexed.

## Database & models

- Soft deletes: default for business-data models (users, subscriptions,
  content); not for pivots or ephemeral tables. Check an existing model
  before deciding for a new one, don't decide ad hoc.
- Soft-deleted data is purged by the `app:purge-soft-deleted` scheduled
  command per each model's retention window — don't write ad-hoc permanent
  deletes; add the model to the purge config instead.
- New tables get a migration + factory + (if exposed via API) a Resource, in
  the same PR.
- **Express query and data-access logic in idiomatic Eloquent — not the
  `DB::` facade or raw SQL.** Use local query scopes (`scopeActive`) for
  reusable constraints, expressive relationships (`hasManyThrough`,
  polymorphic, `whereHas`/`withCount`) for how data connects, and
  accessors/casts for derived attributes. Drop to `DB::`/raw SQL only when
  Eloquent genuinely can't express it — and say why in the PR.
- **Scopes and relationships answer "which rows / how data connects" and live
  on the model; services answer "what to do / whether we should" and hold the
  business rules.** This is complementary to the service-layer rule, not in
  tension with it. A scope that encodes a policy decision (e.g.
  `scopeEligibleForRefund` baking in refund rules) is business logic in
  disguise — keep that in a service. Models stay query-expressive, not fat
  with business rules.
- **Local scopes freely; global scopes deliberately.** A global scope
  silently rewrites *every* query on the model (soft deletes already use one)
  — add another only when that's genuinely intended, mindful of how it
  interacts with the purge and query-builder paths.
- `spatie/laravel-query-builder` allow-lists should point at scopes
  (`AllowedFilter::scope(...)`) so the query-param layer reuses model scopes
  rather than duplicating the same constraints inline.
- **Back fixed-set columns with native PHP enums + Eloquent casts**, not raw
  strings/ints. Status, type, and severity columns get a backed enum (e.g.
  `AnnouncementType`) cast in `casts()`, so the value is type-safe from DB →
  DTO → response. `ApiErrorCode` is the existing example.
- **Mass-assignment is explicit.** Every model declares `$fillable`
  (allow-list, preferred) or `$guarded` — no `$guarded = []` free-for-all. With
  Eloquent strict mode on, filling an attribute outside `$fillable` throws in
  dev rather than silently dropping it.
- **The scheduler must run in every environment.** `app:purge-soft-deleted`
  and any future scheduled command only fire if `* * * * * php artisan
  schedule:run` is in the deploy — the same silent-failure class as a dead
  queue worker. Belongs in the deploy checklist.

## Type strictness & DTOs

- **`declare(strict_types=1)` in every PHP file** — Pint-enforced. Without it,
  typed signatures still silently coerce (`"5"` → `5`, `null` → `0`); this is
  the switch that makes the types real.
- **Full type coverage, no exceptions:** typed properties, parameter types,
  and return types (`: void` / `: never` / `?T` / unions) on everything.
  `mixed` and untyped `array` must be justified. **Larastan runs at `max`
  level in CI — a missing/loose type fails the build.** PHP has no compiler;
  Larastan *is* the compiler here. Treat a Larastan error like a Java/C# type
  error, not a lint nag.
- **Typed objects over associative arrays at every layer boundary.** An
  `['key' => ...]` array crossing controller ↔ service ↔ caller is a smell.
  Use `spatie/laravel-data` DTOs (the app's request-validation +
  service-in/out + API-output object), enums for fixed sets, and Collections
  for lists. Arrays are fine as internal locals — they must not be the
  *contract* between layers.
- **Eloquent strict mode on in non-production.** In
  `AppServiceProvider::boot()`:
  `Model::shouldBeStrict(! $this->app->isProduction())`. Local, staging, and CI
  then throw on lazy-loaded relationships (**catches N+1 in dev**), on filling
  attributes not in `$fillable` (silent-discard), and on accessing
  missing/unloaded attributes (typo/column protection). Production stays
  lenient so an unforeseen lazy-load never 500s a real user. Consequence:
  **eager-load explicitly** (`with` / `load` / `whereHas` / `withCount`) — that
  discipline is the point, not a reason to switch strict mode off. This is the
  closest Laravel gets to compile-time strictness over Eloquent's dynamic
  surface.
- **Scope of the rule:** it governs app code (services, actions, controllers,
  DTOs). It does *not* fight the framework where arrays are the native API
  (config, validation-rule arrays, migration schema builders). Eloquent models
  stay dynamic — express types there via `casts()` and typed accessors, not by
  pretending they're POCOs.

## Queues

- Mail and notifications are **always** `ShouldQueue` — never send email
  (verification, password reset, any notification) synchronously in a
  request cycle.
- The queue runs on **Redis with Horizon** as the worker (adopted
  2026-07-22) — local Redis comes from the Sail compose file, and Predis is
  the client (no phpredis extension assumed). Cache and sessions deliberately
  stay on the database driver; don't move them to Redis without raising it.
- Local mail goes to **Mailpit** (Sail service): SMTP on port 1025, web UI on
  8025. Never point local dev at a real SMTP relay.

## Dev tools & observability

- Telescope ships in **every environment** — never regress it to a dev-only
  package. Access is gated by the `developer` session guard: a separate
  `developers` table + Filament panel at `/devtools`, fully disjoint from app
  users and admins (`is_admin` grants nothing here). Seed the first account
  with `php artisan app:create-developer`.
- `/telescope` (and later `/pulse`, `/horizon`) routes are protected by the
  shared `EnsureDeveloper` middleware — new dev tools reuse it rather than
  inventing a second gate, and get a navigation link in the `/devtools` panel.
- Recording: everything outside production; production records only failures,
  slow queries, and monitored entries. `telescope:prune` is on the scheduler —
  keep it there, or `telescope_entries` grows unbounded.
- **Laravel Boost** (dev-only) is the AI/MCP layer: `.mcp.json` registers
  `boost:mcp` for Claude Code, and per-stack agent skills live in
  `.claude/skills` / `.github/skills` / `.agents/skills`. After adding or
  upgrading a major package, run `php artisan boost:update` so the generated
  `<laravel-boost-guidelines>` block and skills track the real stack. That
  block (appended below this file) is machine-generated and app-specific —
  hand-written conventions stay above it, and the generator repo's copy of
  this file deliberately omits it.
- `composer run dev` is the one-command local start: it brings up the Sail
  services (Redis, Mailpit) and runs the concurrent dev processes. Horizon —
  not `queue:listen` — is the queue worker (Horizon's own `dev` integration
  swaps it in), so local matches production and the /devtools Horizon
  dashboard is live; restart the runner to pick up job-class changes.
- `composer run sync` regenerates the API contract artifacts (Scramble
  `openapi.json` + generated TypeScript types) — run it after route or
  DTO/enum changes; the Bruno collection stays a manual re-sync.
  `php artisan app:fresh` resets the local world (fresh DB + seeds, flushed
  Redis queue/Horizon state, `dev@example.com`/`password` devtools login)
  and refuses to run in production.
- IDE autocomplete comes from `barryvdh/laravel-ide-helper`, regenerated on
  `composer update` (post-update-cmd; meta needs the raised memory limit).
  The three generated files stay gitignored — never commit them.
- `php artisan app:doctor` prints the stack checklist (green check / red X
  per service, tool, and config). **When the stack changes — new service,
  dev tool, or gate — add a matching check to
  `app/Console/Commands/Doctor.php` in the same PR.** Required rows gate the
  exit code (deploy-friendly); optional rows (Xdebug, a running Horizon,
  seeded developer) never do.
- Step debugging is **Xdebug**, with committed `.vscode/launch.json` configs
  (host PHP, Sail container, current Pest file — port 9003). The Sail image
  ships Xdebug (`SAIL_XDEBUG_MODE=develop,debug`); host machines install it
  per-dev (`pecl install xdebug`) with `xdebug.mode=debug,develop` and
  `xdebug.start_with_request=trigger` — trigger mode keeps un-debugged
  requests at full speed. Don't commit machine-specific paths into
  launch.json.
- Pulse (`/pulse`) and Horizon (`/horizon`) are installed behind the same
  `EnsureDeveloper` middleware, with nav links in the `/devtools` panel.
  `horizon:snapshot` runs on the scheduler (queue metrics); Pulse is disabled
  in tests (`PULSE_ENABLED=false`) but its routes stay registered.

## Events & observers

- **Decouple side effects with events + listeners.** When an action has
  secondary effects (registration → welcome notification, subscription
  activated → provisioning), fire a domain event from the service and handle
  each effect in a listener, rather than inlining them. Keeps the primary
  action focused and each effect independently testable.
- **Listeners that do I/O (mail, HTTP, notifications) are `ShouldQueue`** — same
  rule as notifications; never block the request on a listener.
- **Model-lifecycle concerns go in an Observer** (UUID generation, cleanup on
  delete), not scattered `boot()` closures. Observers hold lifecycle mechanics,
  not business policy — same model-vs-service line drawn elsewhere.

## Timezones

**UTC in the core, local only at the edge** — the same convert-at-the-boundary
shape as DTO-in/Resource-out, and the backend-neutral/client-localizes shape
as the API's `error_code` i18n split.

- **`config/app.php` timezone stays `UTC` — never change it.** All storage, all
  Carbon instances, and all service-layer logic are UTC. Datetime columns use
  `datetime` / `immutable_datetime` casts. Changing the app timezone to a local
  zone is the single most common source of timezone bugs — don't.
- **The API emits ISO-8601 with an explicit offset/`Z`**
  (`2026-07-21T10:30:00Z`); **clients localize.** Mobile and Vue render
  device-local; the backend never sends pre-formatted local time (same offload
  as API i18n — backend neutral, client does the last-mile formatting).
- **Input is converted to UTC at the boundary** (Form Request / DTO). Accept
  only unambiguous datetimes — ISO-8601 with an offset/`Z`; reject a naive
  `"2026-07-21 09:00"` with no zone.
- **Convert UTC → local only at the presentation edge** — Blade emails,
  Filament display (`->timezone(...)`). Never in the domain/service layer.
- **Date-boundary queries** ("today", "this month") compute the boundaries in
  the target zone, convert to UTC, then query the UTC range — never compare a
  UTC column to a local date (the classic off-by-a-day bug).
- **A per-user `timezone` column is deferred** (like the per-user locale
  column) — not needed while clients localize. Add it only when a feature
  renders local time *server-side* (scheduled "9am their time" emails,
  server-rendered reports). When added, store IANA names (`Asia/Manila`), never
  fixed offsets (`+08:00` ignores DST).

## Billing

- Billing is **deferred** — don't build it unprompted. When a task requires
  it, it goes behind the app-owned `PaymentGateway` interface (Strategy
  pattern) in the service layer. **Never call Cashier or a provider SDK
  directly from app code, Filament, or controllers** — only from inside a
  gateway adapter.
- The interface must express the app's billing needs, not a provider's API.
  If a Stripe-specific concept (payment_intent, price ID) would appear in the
  interface, that's the signal the abstraction is being broken — keep it in
  the adapter.
- Webhook handlers: verify signature → persist raw event → dispatch a queued,
  idempotent (keyed on provider event ID) entitlement update.

## Infra & platform conventions

- CORS stays locked to own domain / empty by default. Never set
  `allowed_origins => ['*']` on authenticated API routes. Adding a
  cross-origin browser client is a deliberate decision — add the specific
  origin, and flag it.
- `TrustProxies` must stay configured with a CIDR range, never a wildcard.
  If signed URLs 403, rate limiting treats everyone as one IP, or HTTPS
  isn't detected, suspect proxy trust config before anything else.
- Keep `/up` shallow and dependency-free (it's the load balancer's target).
  Dependency checks go in the separate `/health` endpoint.
- User-facing strings go through `__()`, not hardcoded inline — even while
  there's only one locale. Don't build locale middleware, translation files,
  or per-user locale columns unprompted; i18n infrastructure is deferred.
- **`env()` is called only in `config/*` files — never in app code.** App code
  reads `config('...')`. An `env()` call outside config returns `null` the
  moment `php artisan config:cache` runs in production — the classic
  "works locally, null in prod" bug.

## Commits

- **Conventional Commits, enforced by the commit-msg hook:**
  `<type>(<optional scope>): <imperative summary ≤72 chars>`, with types
  `feat fix docs style refactor perf test build ci chore revert`. Scope is
  the area touched (`api`, `devtools`, `auth`, `queue`, `deps`, …).
- **The pre-commit hook runs the CI checks locally** — Pint + Larastan +
  Pest when PHP is staged, ESLint + Prettier checks when JS/TS is staged.
  `--no-verify` is an emergency hatch, not a workflow; CI still enforces
  everything.
- Hooks live in `.githooks/` and activate via
  `git config core.hooksPath .githooks` (done automatically by
  `composer run setup`; a manual step on Path A in the README).

## Testing

- New service methods need a Pest test in the same PR that introduces them.
- New API endpoints need at least one test asserting the response envelope
  shape (not just status code).
- Highest-value coverage is **feature tests** — auth flows, the API envelope,
  and **authorization / 403 paths** (every Policy-gated action). Keep pure unit
  tests for services/Support, but don't skimp the feature layer, where the real
  bugs live.
- Run the full triad — **Pint + Larastan (`max`) + Pest — before every
  commit**, not just in CI. Larastan is the type compiler, so running it
  locally is what actually enforces the strict-typing rules. Don't merge if CI
  is red — no exceptions for "I'll fix it later."

## Auth & security

- Web auth: Sanctum session mode. API/mobile auth: Sanctum token mode
  (`Authorization: Bearer`). Don't mix these — a request to `/api/*` should
  never rely on session cookies.
- **Never build a JWT-style refresh-token system on top of Sanctum.**
  Sanctum's model is simple, revocable, expiring tokens — use the
  `/api/v1/refresh` re-issue pattern already in place, not a second
  refresh-token type. If a task seems to need real OAuth2 refresh-token
  semantics (scopes, third-party API consumers), that's a Passport
  decision, not something to hack into Sanctum — flag it and ask.
- Every login path (password, social, passkey) must go through the shared
  `AuthService` — same find-or-create-user + token issuance logic. Don't
  add a login method that bypasses it.
- New auth methods being rolled out (new social provider, passkeys, any
  new login mechanism) get a Pennant feature flag for staged rollout.
  Per-user settings (2FA on/off, notification preferences, etc.) are a
  settings toggle or a user-table column — not a Pennant flag. If unsure
  which one a new toggle is, ask: "is this a team decision about safety of
  release, or a user's personal choice?"
- Never log or expose raw tokens, card details, or webhook secrets, even in
  debug output.
- Stripe webhook handlers must verify signatures and be idempotent — check
  `app/Http/Controllers/Api/WebhookController.php` (or equivalent) for the
  existing pattern before adding a new webhook handler.

## Authorization

Distinct from authentication above (who you are) — this is what you're allowed
to do.

- **Enforced at the HTTP boundary by default** — the Laravel-idiomatic
  placement. **Policies** (model-scoped) and **Gates** (non-model abilities),
  invoked via `$this->authorize()`, the `can` middleware, or a Form Request's
  `authorize()`. Services assume an already-authorized caller and enforce
  *business invariants* (e.g. "can't cancel an already-cancelled subscription"),
  **not** identity checks — this keeps the authenticated user out of service
  signatures and lets services run from CLI/queue/tests without a fake auth
  context.
- **Every model with ownership/permission semantics gets a Policy.** Rely on
  auto-discovery (`Subscription` → `SubscriptionPolicy`); hand-register only if
  the name breaks convention.
- **Filament `/admin` uses the same Policies** — no separate admin
  authorization path.
- **Policies/Gates are the single source of truth, callable anywhere.** When a
  check is genuinely resource-/data-dependent and can't be decided before the
  service runs, call the *same* policy inside the service (`Gate::allows()` /
  `$user->can()`) — never re-implement the check inline. (This is Laravel's
  analog of .NET's resource-based `IAuthorizationService`; Laravel has no
  native service-layer authz the way Spring's `@PreAuthorize` does.)
- **Non-HTTP callers must consciously authorize.** A service invoked from a
  command, queued job, listener, or Filament action never passes through
  controller/middleware — "it was authorized at the boundary" doesn't apply, so
  decide authorization explicitly there.
- **API:** `AuthorizationException` → `403` + `error_code: UNAUTHORIZED` via the
  centralized handler. Never leak a 403 HTML page.

## When in doubt

- Check `BOILERPLATE.md` for the reasoning behind a decision before
  deviating from it.
- If a task seems to require breaking one of the rules above, say so
  explicitly and ask, rather than quietly working around it.
- Prefer extending an existing service/pattern over introducing a new one.
  This boilerplate's value comes from consistency across projects — a
  clever one-off solution in a single project defeats the point.
