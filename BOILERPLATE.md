# Boilerplate Architecture — Laravel + Inertia SaaS Starter

> Purpose: a generic, reusable starting point for a new indie SaaS idea (not
> client work). Optimized for speed-to-first-user, not feature completeness.
> Cap on initial build time: ~1-2 weeks. Extend only after 2+ real projects
> have repeated the same need.

## Stack summary

| Layer | Choice | Why |
|---|---|---|
| Backend | Laravel (latest) | Existing deep expertise |
| Authenticated app UI | Inertia + Vue 3 + Pinia | Reuses existing Vue/Nuxt skill 1:1, app-like interactivity |
| Public/marketing pages | Blade (+ Alpine.js for light interactivity) | SEO, fast load, no SPA bootstrap cost |
| Admin panel | Filament (Livewire-based) | CRUD-heavy internal tooling, don't reinvent |
| Mobile/API consumers | REST API (`/api/v1`), Sanctum token auth | Client-agnostic — native Swift, cross-platform, or web all hit the same contract |
| Styling | Tailwind CSS | Consistency with Corvin stack |
| Payments | App-owned `PaymentGateway` interface; Stripe (via Cashier) as first adapter | Deferred to when needed; abstracted so providers are swappable |
| Feature flags | Laravel Pennant | Staged rollout, kill-switch for risky features |
| Passkeys | `laravel/passkeys` + Fortify | First-party WebAuthn, behind a Pennant flag |
| Queues | Redis + Horizon (queue only) | Cache/sessions stay on database; Predis client, Redis via Sail locally |

## Core principle: shared service/action layer

Business logic lives in `app/Services/` (or `app/Actions/` for single-purpose
classes), never in controllers. Web (Inertia), Admin (Filament), and API
controllers are all thin — they validate input, call the service, and format
the *output* differently. For Filament this includes Resource, Page, and Action
classes — business logic stays in the service, never in Filament hooks
(`mutateFormDataBeforeSave`, action closures). This guarantees business rules
cannot drift between web, admin, and mobile.

```
Controller (Web/Api/Filament) → Service/Action → Model/DB
                ↓
        Response formatting only (Inertia::render / JSON Resource / Filament UI)
```

**Multi-write operations are atomic and wrapped in `DB::transaction()` inside
the service** — a service method doing more than one write (create-user +
issue-token, entitlement updates) commits all-or-nothing. Transaction
boundaries live in the service layer, never spread across a controller.

## Directory layout

Each kind of class introduced across these conventions has exactly one home,
which is what keeps `app/Support` from becoming a catch-all:

| Directory | Holds |
|---|---|
| `app/Services/`, `app/Actions/` | Business logic — orchestration, domain rules, side effects |
| `app/Data/` | `laravel-data` DTOs (typed layer-boundary payloads) |
| `app/Enums/` | Native PHP enums (`ApiErrorCode`, `AnnouncementType`, …) |
| `app/Support/` | Generic, stateless, domain-agnostic helpers — utility classes, macros, traits, small value objects |
| `app/Policies/` | Authorization policies |

`app/Support` holds only what would be equally at home in *any* Laravel project
— if a class carries domain knowledge or dependencies, it's a Service/Action,
not Support. Helpers are class-based and typed (static-method or invokable
classes); **no global `helpers.php`**, which would be namespace-less,
un-mockable, and invisible to Larastan at `max`. Mirrors `Illuminate\Support`
by design.

## Query & data-access style

Query and data-access logic is expressed in **idiomatic Eloquent** — local
scopes, expressive relationships, accessors/casts — not the `DB::`
query-builder facade or raw SQL (drop to those only when Eloquent genuinely
can't express it, with a stated reason). This is complementary to the service
layer, not in tension with it:

- **Scopes and relationships answer "which rows / how data connects" and live
  on the model.** Encouraged — this is where the framework wants query logic.
- **Services answer "what to do and whether we should" and hold business
  rules.** A scope that encodes a policy decision is business logic in
  disguise and belongs in a service, not the model.

Reach for **local** scopes freely; add a **global** scope only deliberately
(it silently rewrites every query — soft deletes already use one, and a second
one interacts with the purge and query-builder paths).
`spatie/laravel-query-builder` allow-lists reuse these scopes
(`AllowedFilter::scope(...)`) rather than duplicating constraints. The goal:
follow Laravel's own idioms, so the code reads the way the framework intends.

## Type strictness & DTOs

Code is typed as strictly as PHP allows — the target is Java/C#-grade type
safety, reached with strict mode plus static enforcement (PHP has no compiler,
so **Larastan is the compiler** here).

- `declare(strict_types=1)` in every file (Pint-enforced) — without it, typed
  signatures still coerce scalars silently, so the types are a lie.
- Typed properties, parameters, and return types everywhere; `mixed` /
  untyped `array` justified only. **Larastan at `max` level gates CI**, so a
  missing type fails the build rather than rotting in.
- **Typed objects, not associative arrays, across layer boundaries.** DTOs via
  `spatie/laravel-data` — chosen over hand-rolled classes and over Symfony's
  serializer for framework fit: it validates requests into typed objects,
  flows through the service layer and Resources, is Larastan-generics-aware,
  and generates TypeScript types for the Vue app + mobile clients off the same
  definitions. Enums for fixed sets, Collections for lists.
- **Fixed-set columns are backed native PHP enums + Eloquent casts**, not raw
  strings/ints — status/type/severity columns (e.g. announcement
  `type`/`severity`/`dismiss_behavior`) cast to a backed enum so the value is
  type-safe DB → DTO → response. `ApiErrorCode` is the existing example.
- **Mass-assignment is explicit** — every model declares `$fillable`
  (allow-list) or `$guarded`; no `$guarded = []`. Pairs with strict mode below,
  which makes an out-of-`$fillable` fill throw in dev.
- **Eloquent strict mode on outside production** —
  `Model::shouldBeStrict(! app()->isProduction())` in `AppServiceProvider`.
  Dev/staging/CI throw on lazy loading (N+1 caught before prod), silently
  discarded non-fillable attributes, and access to missing/unloaded
  attributes; production stays lenient so an unforeseen lazy-load doesn't 500 a
  live user. The closest Laravel gets to compile-time strictness on Eloquent's
  dynamic surface — the trade is you eager-load explicitly.
- The rule governs app code, not the framework's own array APIs (config,
  validation rules, schema builders); Eloquent models stay dynamic via
  `casts()` / accessors.

## Validation

Input validation lives in **Form Request classes** (`app/Http/Requests`), not
inline `$request->validate()` in controllers — keeps controllers thin and
gives validation one consistent, familiar home. The Form Request owns
`rules()` and, where the endpoint also gates access, `authorize()` (one of the
boundary authorization points above).

The controller then converts the validated data into a typed
`spatie/laravel-data` DTO and passes the **DTO** — not the `$request->
validated()` array — into the service, so the array is absorbed at the
controller edge and the service receives a typed payload.

**Rules live in exactly one place — the Form Request's `rules()`.** The DTO
declares no `rules()` and does not self-validate; it is built from the
already-validated data (`Data::from($request->validated())`) purely as a typed
carrier. Nothing is validated twice and no rule is written twice — the DTO is
transport, the Form Request is the single source of validation truth.

This is a deliberate **two-class split**: Form Request validates, DTO carries
the typed contract. laravel-data *can* collapse both into a single validating
`Data` object, and that's the lower-duplication option — but Form Request
classes are kept here for familiarity and as a dedicated home for complex
conditional validation. (Recorded so a later pass doesn't "consolidate" the
two back into one and think it's an improvement — it's a deliberate trade.)

## Routing & rendering split

- **Guest/marketing routes** (`routes/web.php`, `guest` middleware group) →
  Blade views. Home, pricing, about, blog, login/register (auth pages are a
  judgment call — Blade or Inertia both fine).
- **Authenticated app routes** (`routes/app.php` or similar, `auth` +
  `verified` middleware) → Inertia + Vue pages.
- **Admin routes** (`/admin` prefix) → Filament, fully isolated from Inertia.
  Never mix Livewire components into Inertia pages or vice versa — pick one
  paradigm per page/zone.
- **API routes** (`routes/api.php`, `/api/v1` prefix) → versioned from day
  one, Sanctum token auth (not session), stateless.

## Why Blade for marketing pages (not Inertia)

Inertia pages are client-rendered by default (no SSR without extra
complexity). Marketing pages need: SEO crawlability, fast Core Web Vitals,
reliable Open Graph/meta tags for link previews, and easy A/B testing
tooling — all of which plain server-rendered Blade gives for free. Don't pay
the SPA-bootstrap cost on pages where interactivity doesn't matter.

## Frontend & Inertia conventions

The authenticated UI is Inertia + Vue 3 + Pinia. Inertia controllers are as
thin as API ones — validate (Form Request) → DTO → service → `Inertia::render`
— so web and mobile share the same service and rules can't drift between them.

- **Vue is TypeScript, typed end to end.** `spatie/laravel-data` generates
  TypeScript types from the same DTOs/Resources the backend uses, so page props
  are checked against the backend contract at compile time — the frontend half
  of the strict-typing posture, and the payoff for choosing laravel-data over a
  serializer that can't emit TS.
- **API Resources double as Inertia page props.** The same Resource serializes
  a model for the mobile API and for an Inertia prop — one serialization source
  of truth, identical shapes across clients. (Resolve the Resource so the
  `JsonResource` `data` wrapper doesn't double-nest as a prop.)
- **Global props are shared once** via `HandleInertiaRequests::share()` — auth
  user (minimal, Resource-shaped), flash messages, Pennant flags — not
  refetched per page, kept lean since it ships on every response.
- **Forms use Inertia's `useForm`**; Form Request validation errors flow back
  through Inertia's shared `errors` bag automatically — no hand-rolled fetch or
  error plumbing.
- **UI permission flags derive from the same Policies** (`$user->can(...)`),
  passed as boolean props — the Vue layer only hides/shows, the server still
  enforces (authz stays server-authoritative).
- Persistent layouts (no shell re-mount between visits); lazy/partial props for
  expensive data. The Inertia app is deliberately **not** SSR'd — authenticated
  dashboards aren't indexed (see the decision log).

## Auth strategy

**Two auth modes, one identity system:**

| | Web (Inertia) | Mobile/API |
|---|---|---|
| Mechanism | Sanctum session cookies | Sanctum tokens (`Authorization: Bearer`) |
| Where it lives | HttpOnly cookie | Client-side secure storage |

**Token expiry — Sanctum-native pattern, not JWT-style refresh tokens:**
- Expiry set in `config/sanctum.php` (e.g., 30-90 days) — no separate
  refresh-token type, no custom dual-expiry hacking.
- While a token is still valid, client periodically calls a lightweight
  `/api/v1/refresh` endpoint that issues a fresh token and revokes the old
  one.
- Expired token → 401 → client falls back to full re-login.
- **Explicitly rejected:** hand-rolling OAuth2-style refresh tokens on top
  of Sanctum. Sanctum's design philosophy is simple, revocable, long-lived
  tokens — not short-lived-access + long-lived-refresh. Building the latter
  on top of Sanctum means fighting the package (this is what caused prior
  pain on a client project). If a genuine OAuth2 refresh-token use case
  shows up later (third-party developers consuming the API with scopes),
  that's the trigger to bring in Laravel Passport for that specific case —
  not to retrofit Sanctum.

**Login methods (all funnel into one `AuthService` → same Sanctum token
issuance at the end):**

| Method | Package | Notes |
|---|---|---|
| Email/password | Fortify (built-in) | Baseline |
| Social login | Socialite | Web: OAuth redirect. Mobile: native SDK (Google Sign-In / Sign in with Apple) verifies on-device, ID token sent to backend, backend verifies signature, finds-or-creates user, issues Sanctum token |
| Passkeys | `laravel/passkeys` + Fortify | First-party as of April 2026. Server package + `@laravel/passkeys` npm client (Vue/SSR-safe helpers) + Fortify `Features::passkeys()`. Mobile passkeys use native OS APIs (Sign in with Passkey on iOS, Credential Manager on Android) as the client-side ceremony, same backend verification |
| 2FA (TOTP) | Fortify (built-in) | Per-user opt-in toggle in account settings, not a rollout concern |

**2FA is a challenge step, not a login method.** It doesn't funnel into
`AuthService` alongside the others — it sits *between* a successful credential
check and token issuance. When a user has 2FA enabled, `AuthService` must not
issue the Sanctum token until the TOTP challenge passes; the token-issuance
step is gated on the challenge, not parallel to it.

**Social account-linking rule (account-takeover guard).** Find-or-create by
email is only safe with rules — a provider returning an unverified email, or
silent linking into an existing password account, is a takeover vector. So:
(1) only link/create when the provider asserts `email_verified`; (2) never
auto-link a social identity into an existing password-based account without an
explicit confirmation step. This lives in `AuthService`, once, for every
provider.

**Schema implications:**
- `device_tokens` table (push notifications, even before push is wired up)
- Prefer a separate `social_accounts` table (`provider` + `provider_id`,
  one row per linked identity) over `provider`/`provider_id` columns on
  `users` — multi-provider linking is the case you can't cheaply retrofit, so
  the boilerplate standardizes on it rather than leaving it per-project.
- Apple Sign In included from the start if targeting iOS — Apple requires
  offering it whenever any other third-party login is offered.

## Authorization

Distinct from authentication (above). Enforced at the **HTTP boundary by
default** — Laravel's idiomatic placement — using **Policies** (model-scoped)
and **Gates** (non-model abilities), invoked from controllers
(`$this->authorize()`), the `can` middleware, or Form Request `authorize()`
methods. `/admin` (Filament) reads the *same* Policies, so there's one
authorization definition, not an admin fork.

- Every model with ownership/permission semantics gets a Policy
  (auto-discovered). Services assume an authorized caller and enforce business
  invariants, not identity checks — this keeps the authenticated user out of
  the service signature and lets services run from CLI/queue/tests without a
  fake auth context.
- `AuthorizationException` → `403` + `error_code: UNAUTHORIZED`, JSON envelope,
  via centralized rendering.

**Two guardrails for the one gap this placement has.** Boundary authz doesn't
cover non-HTTP entry points — and Laravel has no *service-layer* authz because
the service layer isn't a framework primitive (Spring's `@PreAuthorize` on a
service method and .NET's injected `IAuthorizationService` both do cover it;
Laravel doesn't). So, by convention:

- **Policies/Gates are the single callable source of truth.** A genuinely
  resource-/data-dependent check that can't be decided before the service runs
  calls the *same* policy inside the service (`Gate::allows()`) — the Laravel
  analog of .NET's resource-based authorization — never a re-implemented inline
  check.
- **Every non-HTTP invocation** (command, queued job, listener, Filament
  action) **must consciously authorize** — "it was authorized at the
  controller" is false for a caller that never touched a controller.

## Feature flags

**Laravel Pennant** — first-party, database-driven by default.

**Decision rule:** is this a *rollout-risk* decision or a *user-preference*
decision?
- **Rollout risk → Pennant feature flag.** Controlled by the team, not the
  user. Use for staged rollout of new/risky functionality (e.g., passkeys —
  new package, staged exposure: internal testers → % rollout → everyone) or
  for the ability to kill a misbehaving feature instantly without a
  redeploy.
- **User preference → a settings toggle / column on the user record, not a
  flag.** Use for things the user chooses for their own account (e.g., 2FA
  enable/disable — this is a per-user choice, not a system rollout
  decision, and Fortify already provides this natively).

Don't default to Pennant for everything — reach for it specifically when
the question is "is this safe to expose to everyone yet," not "does this
user want this on."

## API design standards

**Versioning:** `/api/v1/...` from the start, even as sole consumer. Breaking
changes get a new version, not a breaking change to `v1`.

**Response envelope (success):**
```json
{ "data": { }, "meta": { }, "links": { } }
```
via Laravel API Resources / ResourceCollection — don't hand-roll.

**DTO in, Resource out — one serialization tool per direction.** Inbound, a
Form Request validates and a `spatie/laravel-data` DTO carries the typed
payload into the service; outbound, the service's result is wrapped in an API
Resource. A `Data` DTO is never returned directly as the API response (even
though it can serialize itself) — routing every response through a Resource
keeps the envelope, `whenLoaded` field selection, and versioning uniform. DTOs
go in, Resources come out.

**Response envelope (error):**
```json
{
  "message": "Human-readable, safe to change/localize anytime.",
  "error_code": "SUBSCRIPTION_EXPIRED",
  "errors": { }
}
```
`error_code` is the machine-actionable field clients branch logic on —
`message` is never parsed by clients. Error codes are centralized in an
`ApiErrorCode` enum, documented once.

**HTTP status codes used correctly:**
200 (GET), 201 (created), 204 (deleted, no body), 400 (bad request — e.g.
missing/unknown platform header, `PLATFORM_HEADER_MISSING`), 401
(unauthenticated), 403 (unauthorized), 404 (not found), 422 (validation), 426
(upgrade required — forced app update), 429 (rate limited), 500 (server
error). Every failure
mode returns JSON — never Laravel's HTML debug page — enforced via
centralized exception rendering in `bootstrap/app.php`.

**REST conventions:** plural nouns not verbs (`/subscriptions` not
`/create-subscription`), `apiResource()` routing, max one level of nested
resources, filtering/sorting/pagination via query params
(`?status=active&sort=-created_at`), idempotency awareness on POST for
payment-critical actions.

**Query-param filtering/sorting/includes use `spatie/laravel-query-builder`**
— it implements the `?filter[...]`/`sort=-field`/`include=` grammar once,
consistently, instead of hand-rolling per endpoint. Rules:
- **Allow-lists are mandatory, deny-by-default.** Declare `allowedFilters`,
  `allowedSorts`, `allowedIncludes`, `allowedFields` explicitly on every use.
  This is the point of the package — it stops clients filtering/sorting on
  arbitrary columns (data leak, unindexed-sort perf DoS) or eager-loading
  arbitrary relations. Never expose a bare column set.
- **Read/index controllers only.** Parsing list params is input-shaping, a
  controller concern. Never pass a `QueryBuilder` or the `Request` into a
  service — services take plain scalars, so query-shaping can't smuggle
  business logic back into the controller or the request into the service.
- **Still terminates in `->paginate()` + an API Resource** — the
  never-return-raw-models rule is unchanged.
- **Check the generated `openapi.yaml`:** Scramble can't infer these dynamic
  params, so annotate the filterable/sortable params on documented endpoints.

**Exception — auth & action endpoints are RPC-style by design.** The noun
rule governs *resource CRUD*. Auth and state-transition actions
(`/login`, `/refresh`, `/app-status`, `/announcements/{id}/seen`,
`/announcements/{id}/dismiss`) are intentionally verbs and must not be
"corrected" into contrived resources (`POST /sessions` etc.) by a later
cleanup pass.

**Auth:** see the "Auth strategy" section above. Sanctum token mode for
mobile, session mode for Inertia web.

**App status / version gating / announcements:** see the "App status
endpoint" section below.

**Docs:** Scramble generates OpenAPI 3.1 spec from code (no manual
annotations). Spec exported to a committed `openapi.yaml` / served via route.
Bruno imports the spec into a collection; **OpenAPI Sync** (Bruno) keeps the
collection aligned as the spec evolves, instead of manual re-import. `.bru`
files committed to repo (git-friendly, plain text) so the whole team and
mobile devs always have an up-to-date, importable collection.

## App status endpoint

A single launch-time endpoint plus supporting mechanisms that answer "is it
safe/possible to use the app right now, and is there anything the user
should see." Covers version gating and announcements.

**`GET /api/v1/app-status`** — public (no auth), called once at app launch.
Client sends `X-App-Platform` (ios/android) and `X-App-Version` headers.
Missing/unknown platform → 400 `PLATFORM_HEADER_MISSING`.

```json
{
  "data": {
    "platform": "ios",
    "update_required": false,
    "update_available": true,
    "message": "New version available with performance improvements.",
    "min_supported_version": "1.4.0",
    "latest_version": "1.6.0",
    "store_url": "https://apps.apple.com/..."
  }
}
```

**`data.update_available` here vs. `meta.update_available` elsewhere are two
distinct things, not a contradiction.** On `app-status`, version status *is*
the resource, so it lives in `data`. The `CheckAppVersion` middleware, by
contrast, attaches `meta.update_available` to the envelope of *unrelated* API
responses as a passive soft-nudge. A client reads `data.*` when it explicitly
polls `app-status`, and watches `meta.update_available` on every other call.

### Version gating (two-tier, platform-aware)

Thresholds are **platform-keyed** — iOS and Android release independently and
won't share a version number, so a single global minimum would force you to
gate on the lower of the two. Values live in the `app_settings` table
(key-value, dot-notation keys like `min_supported_version.ios`), managed via
a **Filament Settings page** (fixed set of values, not a CRUD resource),
cached with bust-on-save so the per-request middleware doesn't hit the DB
every call.

| App version vs. thresholds | Behavior |
|---|---|
| `< min_supported_version[platform]` | Hard block — `426` + `error_code: FORCE_UPDATE_REQUIRED`, request not processed |
| `>= min` but `< latest_version[platform]` | Processes normally, response carries `meta.update_available: true` (soft nudge) |
| `>= latest_version[platform]` | Normal response |

**Two enforcement points, same cached source of truth:**
- `GET /app-status` — proactive check at launch, before the user does
  anything (and before login, since it's public).
- `CheckAppVersion` middleware on `api/*` — the enforcement point for
  well-behaved clients; also catches long-running sessions. Emits the 426 hard
  block and sets the `meta.update_available` soft-nudge flag. **Not a security
  control:** because the check fails open on absent headers (below), a client
  that simply omits `X-App-Version` skips the gate. Force-update is a UX
  mechanism — never gate a security fix behind it.
- Store URLs (`store_url.ios/android`) live in the same `app_settings`.

**Fail-open:** if the version/status check itself can't complete (network
failure, missing headers), the client assumes up-to-date and proceeds —
never lock users out on an inconclusive check. Requests without the
version/platform headers (Bruno, partner integrations, web) skip the check
gracefully.

**Client cache:** cache the `app-status` result client-side ~1 hour; always
re-check on cold launch regardless of cache. Endpoint is throttled
(public/unauthenticated).

### Maintenance

Not a custom flag. Rely on Laravel's native `php artisan down` for real
outages — it 503s every route (including `app-status`) and returns JSON to
requests that accept it, with `--secret=` bypass for the team. The mobile
client needs a **global "any request returned 503 → maintenance screen"**
handler, separate from any endpoint-specific logic. A soft
`maintenance_mode` boolean was explicitly considered and rejected — it
doesn't actually stop backend writes, and the "announce ahead / planned
window" case is better served by announcements (below).

### Announcements

A server-authoritative content channel returned in `app-status` (and via a
dedicated endpoint), replacing the rejected maintenance flag. Covers
upcoming maintenance, feature notices, incident banners, general notices —
one mechanism instead of a new field per case.

**Server is the single source of truth for what a user has seen/dismissed.**
The client renders what the server returns and reports interactions back; it
does not run its own dismissal/active-window filtering for logged-in users.
This makes the "reinstall re-bothers the user" and "doesn't sync across
devices" bugs structurally impossible for authenticated users — the install
holds no authoritative state.

**Schema:**
- `announcements` — stable UUID id (never reused), type
  (maintenance/feature/incident/general), severity (info/warning/critical),
  title, body, platform (ios/android/web/all), `dismiss_behavior`
  (permanent/session/none), `starts_at`/`ends_at`, `is_published`.
- Announcements are **not** soft-deleted despite being content — their
  lifecycle is `is_published` + `starts_at`/`ends_at`, so removal is
  unpublish/expire, not delete. This is the deliberate per-model call the
  soft-delete convention asks for, recorded here so it isn't re-decided ad hoc.
- `announcement_user` pivot — `seen_at` + `dismissed_at` per user. `seen_at`
  is included from day one even if unused in v1 UI — it's a one-column cost
  now vs. a painful backfill across every project later, and unlocks
  read/unread UI and "N users saw this" reporting additively.

**Endpoints:**
```
GET  /api/v1/announcements            → active, published, platform-matched,
                                         not-permanently-dismissed for THIS user
POST /api/v1/announcements/{id}/seen
POST /api/v1/announcements/{id}/dismiss
```

**`dismiss_behavior` (server-controlled, set in Filament):**
| Value | Client behavior |
|---|---|
| `permanent` | Server records `dismissed_at`; never shown to that user again (survives reinstall, syncs across devices) |
| `session` | Hidden until relaunch, reappears next cold start until server stops sending it — stays client-side only, nothing to persist |
| `none` | Not dismissible; persistent while active (e.g. critical incident), disappears when server drops it or `ends_at` passes |

**Active-window filtering is server-side** — an announcement only appears
when now is within `starts_at`/`ends_at`, so scheduling "maintenance next
Tuesday" auto-appears and auto-expires without client logic. Managed via a
full **Filament Resource** (it's a growing list of records, unlike version
settings). Announcements are **optional and non-blocking** — a client can
ship without rendering them and nothing breaks; the array is normally empty.

**Logged-out case:** pre-login announcements (auth screen) fall back to
local dismissal, optionally reconciled to the server on login. Rule:
user-scoped server state is authoritative; local storage is only a fallback
for the unauthenticated screen, never the primary store.

**Explicitly out of v1 scope** (additive later, non-retrofit): per-user
targeting (which specific users get an announcement), localization of
announcement text, rich media, read-receipt analytics beyond `seen_at`.

## Queues

Queued by default — running on **Redis with Horizon** as the supervisor
(adopted 2026-07-22; the original database-driver default was upgraded when
Horizon came in). Redis serves the **queue only**: cache and sessions stay on
the database driver, so Redis isn't load-bearing beyond jobs. Local Redis and
Mailpit come from the Sail compose file; the Predis client keeps host PHP
free of the phpredis extension.

- **Mail and notifications are always queued** (`ShouldQueue`), never sent
  synchronously in a request cycle. Email verification and password resets
  are the canonical cases — a request must never block on an SMTP call.
- `failed_jobs` handling + a retry/backoff policy configured from the start.
- **Deploy note (the silent-failure trap):** the Horizon process
  (`php artisan horizon` under Supervisor/systemd) must be running in every
  environment or nothing sends. This belongs in the deploy checklist —
  "works locally, silently broken in prod" is the classic failure here.
- **Deploy note (second silent-failure trap):** the scheduler cron
  (`* * * * * php artisan schedule:run`) must run in every environment, or
  `app:purge-soft-deleted` and every scheduled command silently never fire —
  the same class of bug as a dead worker. Also in the deploy checklist.

## Events, listeners & observers

Secondary effects of an action are decoupled via **events + listeners**, not
inlined in the service: the service does the primary write and fires a domain
event; each side effect (welcome notification on registration, provisioning on
subscription-activated) is a listener. Listeners that do I/O are `ShouldQueue`
(same rule as notifications — never block the request). **Model-lifecycle
mechanics** (UUID generation, cleanup on delete) go in an **Observer**, not
scattered `boot()` closures — and observers hold lifecycle mechanics, not
business policy (the same model-vs-service line drawn elsewhere).

## Soft deletes & scheduled cleanup

- Soft deletes are the default for models representing meaningful business
  data (users, subscriptions, content). **Not** for pure pivot/join tables
  or ephemeral records (sessions, jobs, cache) where it's just noise.
- A scheduled command (`app:purge-soft-deleted`, daily) permanently purges
  soft-deleted rows older than a retention window. The purge dispatches to
  the queue rather than running inline in the scheduler.
- **Retention window is per-model, not one global value** — a deleted user
  may need a 90-day grace (compliance/undo) while deleted draft content is
  fine at 7 days.
- This mechanism also gives GDPR/"right to erasure" a clean path (permanent
  deletion after N days) rather than a manual DB operation, from day one.

## Billing (decided, deferred — NOT a v1 build task)

Billing is **not built in the initial boilerplate**, but the architecture is
decided so it's never accidentally built as direct-Cashier-coupled code:

- App code must **never call Cashier (or any provider SDK) directly.** All
  billing goes through a narrow, app-owned `PaymentGateway` interface in the
  service layer (Strategy pattern), with thin per-provider adapters behind
  it. The Stripe adapter may use Cashier *internally* for subscription/
  webhook/proration heavy-lifting — but that stays inside the adapter.
- The interface expresses *this app's* billing needs (e.g. `createCheckout`,
  `cancelSubscription`, `parseWebhook`), not Stripe's API surface. No
  provider-specific concept (payment_intent, Stripe price IDs) may leak into
  the interface, or the abstraction breaks and a second provider won't fit.
- Which gateway binds is a config value → swapping providers is a container
  binding change, not an app rewrite.
- Webhook handling is uniform across providers: `parseWebhook()` normalizes
  each provider's format into an app-owned `WebhookEvent`; the handler then
  (1) verifies signature, (2) persists the raw event, (3) dispatches queued
  jobs that update entitlements **idempotently** (keyed on provider event ID
  so re-delivery can't double-apply).
- **When actually built:** interface + one real adapter (Stripe via
  Cashier), and stop. No speculative Paddle/PayPal/PayMongo adapters — the
  value is that the seam exists, so the second provider is one adapter class
  against a proven interface, not a re-architecture. (PayMongo noted as
  likely-relevant later given PH market.)

## Health checks

- Shallow `/up` (ships with Laravel 11+) — boots the app, returns 200, no
  dependency checks. This is the **load-balancer health-check endpoint**:
  fast, dependency-free, safe to ping every few seconds.
- Separate deep `/health` endpoint — verifies DB, cache, and queue
  connectivity. For monitoring/alerting to poll at a lower frequency. Keep
  it off the LB's hot path so frequent pings don't hammer the DB.
- **Protect `/health`** (auth, IP allow-list, or a secret query param) — its
  dependency status leaks infra topology, so it should not be world-readable
  like `/up` is.

## CORS

CORS is a browser mechanism — it does not affect native mobile apps (not a
browser) or same-origin Inertia requests, and is independent of the load
balancer. Default posture: **secure by default, open deliberately.**

- `allowed_origins` empty / own-domain only by default. **Never `['*']`** on
  an authenticated API.
- A browser on another origin needing API access (a partner web app, a
  separate frontend domain) is an explicit per-project decision — add that
  origin to the allow-list when the real case appears, not preemptively.

## Load balancer / trusted proxies

If deployed behind a load balancer (or any reverse proxy), configure
Laravel's `TrustProxies`. Without it, every request appears to originate from
the LB's IP — which silently breaks rate limiting (everyone shares an IP),
`$request->ip()`, HTTPS detection (LB terminates SSL, app thinks it's HTTP),
and **signed URLs** (the 403 Invalid Signature class of bug).

- Trust the LB's IP range via **CIDR, not a wildcard** (security).
- This is invisible in local dev (no proxy) and only surfaces in
  production — so it's baked in from the start rather than debugged live per
  project.
- The shallow `/up` route (above) is the LB's health-check target.

## Internationalization (i18n)

Real i18n infrastructure is **deferred** — but the expensive-to-retrofit
parts are already handled cheaply by existing decisions, so keep the door
open at near-zero cost:

- User-facing strings route through Laravel's `__()` translation helpers,
  not hardcoded inline — a cheap habit that makes later translation a
  content task, not a refactor.
- The API's `error_code` / `message` split (already decided) means **clients
  localize their own UI off the code** — the backend never needs to be
  multilingual. This offloads the hardest part of API i18n entirely.
- **Deferred** (add only when a project needs it): locale-detection
  middleware, `Accept-Language` handling, per-user locale preference column,
  actual translation files beyond the default locale.

## Timezones

Same shape as the i18n decision above: **UTC in the core, local only at the
edge; the backend stays timezone-neutral and clients localize.**

- Storage, Carbon instances, and service logic are always **UTC** —
  `config/app.php` timezone stays `UTC` and is never changed to a local zone
  (the single most common timezone bug). Datetime columns use `datetime` /
  `immutable_datetime` casts.
- The API emits **ISO-8601 with an explicit offset/`Z`**; mobile and Vue render
  device-local. The backend never sends pre-formatted local time — the same
  client-localizes offload as the `error_code`/`message` split.
- Convert **to UTC at input** (Form Request / DTO; reject naive datetimes with
  no zone) and **to local only at the presentation edge** (Blade emails,
  Filament `->timezone(...)`) — never in the domain layer.
- Date-boundary queries compute the window in the target zone, convert to UTC,
  then query the UTC range — never compare a UTC column to a local date.
- A **per-user `timezone` column is deferred** (mirrors the deferred per-user
  locale column) — unneeded while clients localize; add it only when a feature
  renders local time server-side, storing IANA names (never fixed offsets,
  which ignore DST). Keeping UTC-core + ISO-8601 API now makes that addition
  additive, not a refactor.

## Config & environment

`env()` is read **only in `config/*` files**; app code reads `config('...')`.
An `env()` call anywhere outside config returns `null` the moment
`php artisan config:cache` runs (which production should) — the canonical
"works locally, null in prod" bug. Baked in as a rule so it's never
introduced.

## Testing & CI

- Pest for feature/unit tests. Baseline coverage required in the boilerplate
  itself: auth flows, service layer methods, API response shape assertions.
- GitHub Actions: run tests + Pint (style) + Larastan/PHPStan (static
  analysis) on every PR.

## Production readiness checklist

- [ ] Error monitoring: deferred (no budget). When added, install the
      **Sentry Laravel SDK but point its DSN at self-hosted GlitchTip**
      (Sentry-API compatible → swap to hosted Sentry later via one env var,
      no code change). **Laravel Pulse** (free, first-party, in-app metrics)
      is the day-one baseline when monitoring is wanted. Telescope runs in
      **every environment** behind the `/devtools` developer panel (separate
      `developers` table + guard); production records only failures, slow
      queries, and monitored entries, with `telescope:prune` scheduled.
- [ ] Horizon running in every environment (`php artisan horizon` under
      Supervisor/systemd) — mail/notifications are queued on Redis, so a
      dead Horizon means nothing sends.
- [ ] Billing: deferred. When built, goes behind the app-owned
      `PaymentGateway` interface — never direct Cashier. Webhook handler
      verifies signature + is idempotent on provider event ID.
- [ ] `/up` shallow health check confirmed (LB target); deep `/health`
      (DB/cache/queue) present for monitoring.
- [ ] `TrustProxies` configured with the load balancer's CIDR range (not a
      wildcard) — prevents broken rate limiting, IP detection, HTTPS
      detection, and signed-URL 403s behind a proxy.
- [ ] Rate limiting (`throttle:api`) on all API routes.
- [ ] `device_tokens` table present even if push notifications aren't wired
      up yet — cheaper to add now than backfill later.
- [ ] CORS locked down by default (own domain / empty), never `['*']` on an
      authenticated API; origins added deliberately per project.
- [ ] User-facing strings go through `__()` (keeps i18n a later content task,
      not a refactor); real i18n infra deferred.
- [ ] Soft-delete convention applied per model + `app:purge-soft-deleted`
      scheduled cleanup running (queued), per-model retention windows.

## Explicit non-goals (don't over-build these upfront)

- No speculative multi-tenancy — add only when a project needs it.
- No i18n scaffolding until a real project requires it.
- No custom admin UI — Filament covers this.
- No hand-rolled API documentation — Scramble/OpenAPI generates it.

## Decision log (why, not just what)

- **Inertia over Nuxt:** one deploy target instead of two, no
  duplicate API layer for the frontend's own use, reuses existing Vue/Pinia
  skill directly. Traded away: SSR for the app itself (not needed —
  authenticated dashboards aren't indexed).
- **Filament kept despite being Livewire-based:** it's isolated to `/admin`,
  doesn't touch Inertia's routes or DOM, and is the right tool for
  CRUD-heavy internal tooling. Coexistence is standard practice, not a
  paradigm conflict, as long as the `/admin` boundary is respected.
- **REST API kept client-agnostic:** no assumptions about native vs.
  cross-platform mobile — API only knows HTTP, JSON, and its own contract.
- **Sanctum over Passport, no custom refresh-token system:** the boilerplate
  is for first-party apps (own web + own mobile), which is exactly Sanctum's
  target use case. Passport's OAuth2 complexity (refresh tokens, scopes,
  client registration) is only justified once third-party developers
  consume the API — premature otherwise, and a past attempt to bolt
  refresh-token semantics onto Sanctum directly caused implementation pain.
- **Passkeys behind a Pennant flag, 2FA not:** passkeys are a new
  first-party Laravel package (April 2026) — staged rollout de-risks
  adoption. 2FA is a per-user setting already provided by Fortify, and
  gating it with a flag would conflate a user preference with a rollout
  decision.
- **Version gating is platform-keyed, two-tier:** iOS/Android release
  independently, so per-platform thresholds are required — a global minimum
  would gate on the lower version. Hard block (426) + soft nudge
  (`meta.update_available`). Managed in Filament so a bad build can be
  force-updated without a deploy.
- **No custom `maintenance_mode` flag:** `php artisan down` handles real
  outages better (atomic, blocks writes); the planned/advance-notice case is
  served by announcements instead. A soft boolean would add a second
  "are we down" concept that doesn't actually stop anything.
- **Announcements are server-authoritative, built full in v1:** for a
  boilerplate every project inherits, dismissal state lives server-side
  (keyed to user, not install) from the start — so reinstall/multi-device
  re-annoyance is structurally impossible and no project has to retrofit
  dismissal-sync later. `seen_at` included day one to avoid a future
  backfill. Targeting/localization deliberately deferred as additive,
  non-retrofit features.
