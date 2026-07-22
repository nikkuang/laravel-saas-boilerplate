# BUILD_PLAN.md — Instructions for Claude Code

> Run this once to scaffold the boilerplate repo from scratch, or re-run
> individual phases to update it. Read `BOILERPLATE.md` first for the
> reasoning behind every decision below — this file is the task list, that
> file is the "why."

## How to use this file

Work phase by phase. After each phase, run tests/lints before moving on.
Don't skip ahead — later phases assume earlier ones are in place. If a
decision here conflicts with something discovered mid-build, stop and ask
rather than silently deviating.

---

## Phase 0 — Project init

- [ ] **Decide first — start from an official starter kit, or assemble from
      parts.** Jetstream (Inertia+Vue stack) / Laravel's current Inertia-Vue
      starter already scaffolds Fortify + Sanctum + 2FA + the action pattern +
      Tailwind — i.e. most of Phase 1–2. **Recommended:** start from the kit and
      conform it to these conventions rather than hand-wiring auth. If
      assembling manually instead, the `laravel new` + install steps below
      stand as-is. This choice changes how much of Phase 1–2 is already done.
- [ ] `laravel new` (latest LTS/stable version)
- [ ] Install (the deliberate additions beyond the starter kit): Inertia
      (Vue 3 adapter), Pinia, Tailwind CSS, Sanctum, Filament, Scramble, Pest,
      Larastan, Laravel Pint, Laravel Pennant, Laravel Socialite,
      `laravel/passkeys`, `laravel/wayfinder` (+ `@laravel/vite-plugin-wayfinder`
      — typed route helpers shared with the Vue frontend), `firebase/php-jwt`
      (verify native Apple/Google **ID tokens** via JWKS in the mobile social
      flow — see Phase 2), `spatie/laravel-data` (typed DTOs — see "Type
      strictness & DTOs"), TypeScript + `spatie/laravel-typescript-transformer`
      (generates TS types from DTOs/Resources for the Vue side)
      (NOT Cashier — billing is deferred, see Phase 6.5)
- [ ] The Laravel Vue starter kit supplies the rest, deliberately **not**
      re-listed above: the Vite/TypeScript toolchain, Blade/Alpine marketing
      deps, the shadcn-vue UI libraries (reka-ui, cva, clsx, tailwind-merge,
      lucide, sonner, vue-input-otp), and the default dev deps (Pest/PHPUnit,
      Faker, Mockery, Collision, Pail, Tinker). If assembling manually instead
      of from the kit, install those too — don't rely on this list for them.
- [ ] Set up `.env.example` with all required keys documented (DB, mail,
      social login credentials, etc. — no Stripe/Sentry yet, both deferred).
      **Database is PostgreSQL, not SQLite** — dev/CI/prod all run Postgres so
      there's no driver skew (SQLite's single-writer locking caused
      intermittent cached-object corruption under concurrent
      web+worker+Pulse access). `DB_CONNECTION=pgsql`; keep `.env`
      host-oriented (`127.0.0.1`) and let the `laravel.test` compose service
      override `DB_HOST`/`REDIS_HOST`/`MAIL_HOST` to the service names, so one
      `.env` works both host-run (Path A) and container-run (Path B) —
      Laravel's immutable dotenv won't overwrite a real env var.
- [ ] Configure Pint (enforce `declare(strict_types=1)` + native type hints)
      + Larastan at **`max` level** — a missing/loose type fails CI. Full type
      coverage and typed DTOs over associative arrays at layer boundaries (see
      "Type strictness & DTOs")
- [ ] Eloquent strict mode in `AppServiceProvider::boot()`:
      `Model::shouldBeStrict(! $this->app->isProduction())` — dev/staging/CI
      throw on lazy loading (N+1), non-fillable attribute fills, and missing
      attribute access; production stays lenient
- [ ] Confirm `/up` shallow health check works (LB target)

## Phase 1 — Routing, rendering split & infra

- [ ] `routes/web.php` — guest group (marketing, Blade) + auth pages
- [ ] `routes/app.php` — authenticated Inertia routes (`auth`,`verified`
      middleware)
- [ ] `routes/api.php` — `/api/v1` prefix, `auth:sanctum` group
- [ ] Filament installed on `/admin`, confirm it boots independently of
      Inertia routes. **Move the default panel's resources/pages under
      `app/Filament/Admin/`** (namespace `App\Filament\Admin\…`, and point
      the panel provider's `discoverResources`/`discoverPages`/`discoverWidgets`
      there) so it isn't the odd one out once a second panel exists — one
      directory per panel from the start
- [ ] Admin bootstrapping: Filament panel-access check (`canAccessPanel()` /
      gate — who is allowed into `/admin`) + a seeder for the first admin user
- [ ] Verify: guest → Blade page loads with zero Vue/Inertia JS in the
      network tab

**Inertia (authenticated UI) conventions:**
- [ ] Inertia controllers thin: Form Request → DTO → service →
      `Inertia::render` (same service layer as API; no logic in the controller)
- [ ] Vue app in TypeScript; `spatie/laravel-typescript-transformer` generates
      TS types from DTOs/Resources so props are type-checked against the backend
- [ ] API Resources reused as Inertia page props (resolve to avoid the
      `data`-wrapper double-nesting) — one serialization source for web + mobile
- [ ] Global props via `HandleInertiaRequests::share()` (minimal auth user,
      flash, Pennant flags), kept lean
- [ ] Forms use Inertia `useForm`; Form Request errors surface via the shared
      `errors` bag (no hand-rolled fetch)
- [ ] UI permission flags from Policies passed as props (server still enforces)
- [ ] Persistent layouts; lazy/partial props for expensive data; app not SSR'd
- [ ] `TrustProxies` configured for a CIDR range (env-driven), NOT a
      wildcard — prevents broken rate limiting / IP / HTTPS detection /
      signed-URL 403s behind a load balancer
- [ ] Deep `/health` endpoint checking DB + cache + queue connectivity
      (separate from the shallow `/up` LB target). **Protect it** (auth / IP
      allow-list / secret) — it leaks infra topology, unlike public `/up`
- [ ] CORS config: locked to own domain / empty by default, never `['*']`
      on authenticated API routes
- [ ] Adopt `__()` for user-facing strings from the start (i18n door open);
      do NOT build locale middleware / translation files / per-user locale
      yet
- [ ] `env()` only in `config/*` files; app code reads `config()` — an
      `env()` call outside config returns null under `config:cache` in prod
- [ ] Timezones: keep `config/app.php` timezone `UTC` (never change it); store
      all datetimes UTC; API emits ISO-8601 with offset/`Z` and clients
      localize; convert to UTC at input, to local only at display. Per-user
      `timezone` column deferred (IANA names when added — never fixed offsets)

## Phase 2 — Auth

- [ ] Sanctum configured for **both** modes: session (web/Inertia) and
      token (API/mobile)
- [ ] Login/register pages (Blade or Inertia — pick one, be consistent)
- [ ] Token issuance endpoint for mobile (`POST /api/v1/login` →
      `createToken()`)
- [ ] Set token expiry in `config/sanctum.php` (30-90 days). Build
      `POST /api/v1/refresh` — issues a fresh token + revokes the old one,
      while current token is still valid. Expired token → 401 → client
      re-logs in. **Do not build a separate refresh-token type or dual
      expiry system** — see BOILERPLATE.md "Auth strategy" for why.
- [ ] `device_tokens` migration (even if push isn't wired up yet)
- [ ] `social_accounts` table (`provider` + `provider_id`, one row per linked
      identity) — standardized over columns on `users`; multi-provider linking
      is the case you can't cheaply retrofit
- [ ] Socialite installed. Web: standard OAuth redirect flow (Google at
      minimum). Mobile: endpoint(s) that accept a native SDK's ID token,
      verify signature, find-or-create user, issue Sanctum token
- [ ] Apple Sign In included if targeting iOS at all
- [ ] **Verify first:** confirm `laravel/passkeys`, Fortify
      `Features::passkeys()`, and the `@laravel/passkeys` npm client actually
      exist at the assumed versions before relying on them. This plan assumes a
      first-party passkeys package (April 2026) — if the real package/API
      differs, **STOP and ask**, don't silently work around it.
- [ ] `laravel/passkeys` + Fortify installed, `Features::passkeys()`
      enabled, `@laravel/passkeys` npm client wired into the Vue login
      component. **Wrap passkey login option in a Pennant feature flag**
      (start narrow — internal testers only — widen later)
- [ ] Fortify 2FA (TOTP) scaffolding enabled as a per-user settings toggle
      — **not** behind a feature flag (see BOILERPLATE.md "Feature flags"
      for the flag-vs-preference distinction)
- [ ] All of the above (password, social, passkey login) funnel into one
      shared `AuthService` that does find-or-create-user + Sanctum token
      issuance — confirm no login path duplicates this logic. **2FA is a
      challenge step, not a funnel input** — gate token issuance on the TOTP
      challenge, don't issue before it
- [ ] Dedicated brute-force throttle on `/login`, `/refresh`, and the
      social/passkey callbacks (keyed on email+IP) — tighter than the general
      `throttle:api`
- [ ] Social account-linking guard in `AuthService`: only link/create on
      provider `email_verified`; never auto-link a social identity into an
      existing password account without an explicit confirmation step
      (account-takeover vector). Use a `social_accounts` table (one row per
      linked identity), not `provider`/`provider_id` columns on `users`
- [ ] Verify: end-to-end test for at least one full login path (password)
      and confirm token issuance shape matches the API envelope standard

**Authorization (distinct from the authentication above):**
- [ ] Policies for every model with ownership/permission semantics
      (auto-discovered); Gates for non-model abilities. Enforce at the HTTP
      boundary — `$this->authorize()` / `can` middleware / Form Request
      `authorize()`. Services assume an authorized caller (business invariants
      only, not identity checks)
- [ ] Filament `/admin` uses the same Policies — no separate admin authz path
- [ ] Guardrails: Policies/Gates are the single callable source of truth — a
      resource-/data-dependent check calls the same policy inside the service
      (never re-implemented); every non-HTTP caller (command/job/listener/
      Filament action) authorizes consciously
- [ ] `AuthorizationException` → `403` + `error_code: UNAUTHORIZED`, JSON
      envelope (wired in Phase 4's centralized handler)

## Phase 3 — Service/Action layer convention

- [ ] Create the directory layout: `app/Services/` + `app/Actions/` (business
      logic), `app/Data/` (laravel-data DTOs), `app/Enums/` (native enums),
      `app/Support/` (generic class-based helpers/macros/traits — no global
      `helpers.php`), `app/Policies/`
- [ ] Build one full example vertical slice end-to-end to prove the pattern:
      e.g. a `SubscriptionService` used by a Web Inertia controller, an API
      controller, and a Filament resource — same service, three response
      shapes
- [ ] Confirm: no business logic exists in any controller, only in services
- [ ] Multi-write service methods wrap in `DB::transaction()` (atomic);
      transaction boundaries live in the service, not controllers
- [ ] Decouple secondary effects with events + listeners (I/O listeners
      `ShouldQueue`); model-lifecycle mechanics in Observers, not `boot()`
      closures

## Phase 4 — API standards

- [ ] API Resources for every model exposed via API (never return raw
      models)
- [ ] `spatie/laravel-query-builder` installed; list/index endpoints use it
      for filtering/sorting/includes with **mandatory allow-lists**
      (`allowedFilters`/`allowedSorts`/`allowedIncludes`/`allowedFields`),
      read-controllers-only, terminating in `->paginate()` + a Resource.
      Annotate dynamic params for Scramble
- [ ] Input validation in Form Request classes (`rules()` + `authorize()`),
      never inline `$request->validate()`. Controller converts validated data
      into a `spatie/laravel-data` DTO and passes the DTO (not the array) to
      the service. **Rules live only in the Form Request** — the DTO carries no
      `rules()` and doesn't self-validate (`Data::from($request->validated())`),
      so no rule is duplicated
- [ ] Centralized exception handling in `bootstrap/app.php` — guarantee
      JSON responses for every error path on `api/*` routes (validation,
      authentication, authorization/403, 404, 500)
- [ ] `ApiErrorCode` enum created with at least the common cases
      (UNAUTHENTICATED, UNAUTHORIZED → 403, VALIDATION_FAILED, NOT_FOUND,
      RATE_LIMITED, PLATFORM_HEADER_MISSING → 400, FORCE_UPDATE_REQUIRED →
      426) — extend per project; no inline error-code strings anywhere
- [ ] `throttle:api` rate limiting applied
- [ ] Scramble installed and generating OpenAPI 3.1 spec from existing
      routes with zero manual annotation
- [ ] Export spec to `openapi.yaml`, committed to repo
- [ ] CI step that regenerates the spec and fails on a diff against the
      committed `openapi.yaml` — a committed generated file drifts silently
      otherwise
- [ ] Bruno collection created via OpenAPI import, OpenAPI Sync connected,
      `.bru` files committed to repo

## Phase 4.5 — App status (version gating + announcements)

Read the "App status endpoint" section of BOILERPLATE.md before building —
several decisions here are deliberate and easy to get subtly wrong.

**Version gating:**
- [ ] `app_settings` key-value table + cached accessor (bust-on-save)
- [ ] Seed default `app_settings` values (min/latest version + store URLs per
      platform) so the Filament Settings page has editable defaults on a fresh
      install
- [ ] Filament **Settings page** (not a resource) for platform-keyed values:
      `min_supported_version.{ios,android}`, `latest_version.{ios,android}`,
      `store_url.{ios,android}`
- [ ] `CheckAppVersion` middleware on `api/*`: reads `X-App-Platform` +
      `X-App-Version`; `426 FORCE_UPDATE_REQUIRED` below min; sets
      `meta.update_available` between min and latest; **skips gracefully**
      when headers absent (web/Bruno/partners)
- [ ] `GET /api/v1/app-status` — public, throttled, platform-aware; returns
      version status + store URL + active announcements; `400
      PLATFORM_HEADER_MISSING` on missing/unknown platform
- [ ] Add `426` to the `ApiErrorCode` / status handling
- [ ] Confirm fail-open behavior is documented for the client (never lock
      out on an inconclusive check)

**Maintenance:**
- [ ] Confirm `php artisan down` returns JSON to `Accept: application/json`
      requests; document the client's required global 503 handler. **Do
      not** build a custom `maintenance_mode` flag.

**Announcements (server-authoritative — build in full, not a stub):**
- [ ] `announcements` table (UUID pk, type, severity, title, body, platform,
      `dismiss_behavior`, `starts_at`, `ends_at`, `is_published`)
- [ ] `announcement_user` pivot with **both** `seen_at` and `dismissed_at`
      (include `seen_at` even if v1 UI doesn't use it — avoids a backfill)
- [ ] `GET /api/v1/announcements` — active + published + platform-matched +
      not-permanently-dismissed-for-this-user (server-side filtering, incl.
      `starts_at`/`ends_at` window)
- [ ] `POST /api/v1/announcements/{id}/seen` and `/dismiss`
- [ ] `dismiss_behavior` handling: `permanent` (persist server-side),
      `session` (client-only), `none` (not dismissible)
- [ ] Filament **Resource** (full CRUD) for announcements — create/schedule/
      publish, set type/severity/platform/dismiss_behavior
- [ ] Logged-out fallback: local dismissal on pre-login screen, optional
      reconcile-on-login. Server state authoritative wherever a user exists.
- [ ] **Out of scope for v1** (do not build): per-user targeting,
      localization, rich media, analytics beyond `seen_at`

## Phase 5 — Marketing pages

- [ ] Blade layout for public pages (home, pricing, about) with per-page
      meta/OG tag support
- [ ] Alpine.js wired in for lightweight interactivity (waitlist form, nav
      toggle) — confirm no Livewire or Inertia dependency leaks in here
- [ ] Confirm signup form submits and correctly redirects into the Inertia
      app post-registration (no double-bounce through an intermediate page)

## Phase 6 — Queues & soft deletes

**Queues:**
- [ ] Redis queue driver + Horizon (Predis client — no phpredis extension
      assumed), `failed_jobs` table + retry/backoff policy; cache and
      sessions stay on the database driver
- [ ] Mark mail + notifications `ShouldQueue` — email verification and
      password reset must be queued, not synchronous
- [ ] Deploy docs note: Horizon MUST run in every environment or nothing
      sends (`php artisan horizon` under Supervisor/systemd)
- [ ] Deploy docs note: the scheduler cron (`* * * * * php artisan
      schedule:run`) MUST run in every environment, or `app:purge-soft-deleted`
      (and all scheduled commands) silently never fire

**Soft deletes:**
- [ ] Soft deletes on business-data models (users, subscriptions, content);
      NOT on pivots/ephemeral tables
- [ ] Per-model retention window (config or model property), not one global
      value
- [ ] `app:purge-soft-deleted` scheduled command (daily) that dispatches a
      queued job to permanently delete rows past their retention window

**Model conventions:**
- [ ] Back fixed-set columns (status/type/severity, incl. announcement
      `type`/`severity`/`dismiss_behavior`) with native PHP enums + `casts()`
- [ ] Declare `$fillable`/`$guarded` explicitly on every model (no
      `$guarded = []`)

## Phase 6.5 — Billing (DEFERRED — do not build in v1)

Billing is **decided but intentionally not built** in the initial
boilerplate. Do NOT install Cashier or wire payments during the initial
build. When a project actually needs billing, build it per the "Billing"
section of BOILERPLATE.md:
- App-owned `PaymentGateway` interface (never call Cashier/SDK directly from
  app code)
- One real adapter only (Stripe, may use Cashier internally), behind the
  interface
- Uniform webhook handling: verify → persist raw event → queued idempotent
  entitlement update
- Stop at one provider; no speculative adapters

## Phase 7 — Testing & CI

- [ ] Pest baseline tests: auth flow, one full service-layer test, API
      response shape assertion for at least one endpoint, and an
      authorization / 403 assertion for a Policy-gated endpoint
- [ ] Run Pint + Larastan (`max`) + Pest before commit (optionally a
      pre-commit hook) — Larastan locally enforces the strict-typing rules
- [ ] GitHub Actions workflow: run Pest + Pint + Larastan on every PR.
      **Tests run on PostgreSQL, not SQLite in-memory** (dev/CI/prod
      parity — catches driver-specific issues): phpunit points at a
      `testing` database; CI adds a `postgres:17` service and creates
      that database. Slower than :memory:, deliberately.
- [ ] Committed `.githooks/`: pre-commit (Pint --dirty + Larastan + Pest when
      PHP staged; ESLint/Prettier checks when JS/TS staged) and commit-msg
      (Conventional Commits regex, merge/fixup exempt). Activate via
      `git config core.hooksPath .githooks` in `composer run setup` and as a
      manual README step; document `--no-verify` as emergency-only
- [ ] DX commands: `composer run sync` (scramble:export +
      typescript:transform), `app:fresh` local reset (refuses production;
      seeds dev@example.com devtools login), and
      `barryvdh/laravel-ide-helper` on post-update-cmd (meta needs
      `-d memory_limit=512M`; generated files gitignored). Doctor rows for
      hooks + ide-helper
- [ ] Confirm CI fails on a deliberately broken test (sanity check the
      pipeline actually works)

## Phase 8 — Observability & dev tools

- [ ] `composer require laravel/telescope` as a **production dependency**
      (not `--dev`): Telescope ships in every environment, gated by auth —
      never dev-only. `telescope:install` + migrate. Keep
      `TELESCOPE_ENABLED=true` in `phpunit.xml` (routes only register when
      enabled, and the route-protection tests need them; the recording
      filter returns false under tests so nothing is stored).
- [ ] Devtools surface: `developers` table + `Developer` model (no soft
      deletes — deleting a developer is an access revocation and must be
      immediate) + `developer` session guard/provider in `config/auth.php`,
      then a **second Filament panel** at `/devtools`
      (`->authGuard('developer')`) under `app/Filament/Devtools/` (its own
      directory + `App\Filament\Devtools\…` namespace, discovery paths
      isolated from Admin's), with a Developers CRUD resource and external nav
      links to the tools. Completely disjoint from app users/admins —
      `is_admin` grants nothing here.
- [ ] `EnsureDeveloper` middleware (redirects to `/devtools/login` unless
      `auth('developer')->check()`) replaces Telescope's stock `Authorize`
      in `config/telescope.php`. Pulse and Horizon reuse this same
      middleware when added — one gate for all dev tools.
- [ ] Recording filter in `TelescopeServiceProvider`: everything outside
      production; production keeps only exceptions, failed requests/jobs,
      slow queries, scheduled tasks, and monitored tags. Schedule
      `telescope:prune` daily (staging records everything — pruning is what
      bounds the table).
- [ ] `app:create-developer` artisan command (Laravel Prompts) to seed the
      first developer per environment.
- [ ] Password reset on both panels: `->passwordReset()` on each; the
      `/devtools` panel adds `->authPasswordBroker('developers')` with a
      `developers` broker (config/auth.php) + a dedicated
      `developer_password_reset_tokens` table (separate from app users' so a
      shared email can't collide). The Developer model already satisfies
      `CanResetPassword` via `Foundation\Auth\User` + `Notifiable`; Filament's
      reset notification is `ShouldQueue`. Test the broker isolation.
- [ ] Local services via Sail: `php artisan sail:install --with=pgsql,redis,mailpit`
      publishes the compose file (PostgreSQL 17 + Redis + Mailpit, ports
      forwarded to localhost; the pgsql service mounts Sail's
      create-testing-database.sql so a `testing` DB exists for the suite).
      **Bring services up with `sail up -d --wait`** (not bare `-d`) in the
      setup/dev scripts and give Postgres a healthcheck with a `start_period`
      — `up -d` returns when the container *starts*, not when Postgres accepts
      connections, so a bare `-d` races `migrate` against first-boot initdb and
      fails with "server closed the connection unexpectedly".
      `.env` points the host at `127.0.0.1`; the `laravel.test` service
      overrides `DB_HOST`/`REDIS_HOST`/`MAIL_HOST` to the Docker service names
      so container-run commands (`./vendor/bin/sail artisan`) connect too.
      `MAIL_MAILER=smtp` port 1025, Mailpit UI on 8025, `REDIS_CLIENT=predis`.
      Watch installer collateral: `horizon:install` strips
      `declare(strict_types=1)` from `bootstrap/providers.php`, and
      `sail:install` clobbers the phpunit DB env — restore both.
- [ ] `composer require laravel/pulse`, publish Pulse, migrate (Horizon +
      Predis were installed in Phase 6). Both dashboards go behind
      `EnsureDeveloper` via their
      config `middleware` arrays; the `viewHorizon` gate checks the
      `developer` guard; nav links added to the /devtools panel;
      `horizon:snapshot` scheduled every five minutes; `PULSE_ENABLED=false`
      in phpunit (recording off, routes stay registered).
- [ ] Feature tests: guests, app users, and `/admin` admins are all
      redirected off `/telescope` and `/devtools`; a `developer`-guard
      session gets 200. (Note: `actingAs($dev, 'developer')` also switches
      the default guard — reset with `auth()->shouldUse('web')` to mirror
      real requests.)
- [ ] `app:doctor` command: ✓/✗ checklist of the whole stack (PHP version,
      APP_KEY, config cache, DB, Redis, queue, SMTP/Mailpit, Telescope,
      Pulse, Horizon installed+running, Xdebug, seeded developer). Required
      rows drive the exit code; keep its checks in sync with stack changes.
- [ ] Step debugging: commit `.vscode/launch.json` (host + Sail-container +
      Pest-file configs, port 9003; carve it out of .gitignore with
      `/.vscode/*` + `!/.vscode/launch.json`). Sail's image ships Xdebug
      (`SAIL_XDEBUG_MODE=develop,debug` in .env); host machines install
      per-dev via `pecl install xdebug` with `xdebug.mode=debug,develop` +
      `xdebug.start_with_request=trigger`.
- [ ] `composer require laravel/boost --dev` +
      `php artisan boost:install --guidelines --skills --mcp -n`: registers
      the Boost MCP server (.mcp.json) and generates version-pinned AI
      guidelines/skills for detected agents. The `<laravel-boost-guidelines>`
      block it appends to CLAUDE.md is machine-generated — keep hand-written
      conventions above it (the generator's CLAUDE.md copy omits the block).
      Re-run `boost:update` after major package changes. Add the optional
      doctor row.
- [ ] Error monitoring (Sentry/GlitchTip) is **deferred (no budget)** — do
      not install/wire it in v1. When added: Sentry Laravel SDK with DSN
      pointed at self-hosted GlitchTip (swap to hosted later via env var, no
      code change). Pulse (in-app metrics) and Horizon (queue monitoring) are
      already installed above behind `/devtools` — this defers only external
      error tracking.
- [ ] Confirm Horizon surfaces `failed_jobs` and queue throughput (its
      dashboard is the monitoring surface; a silently dead Horizon is the
      failure mode to catch)

## Phase 9 — Final pass

- [ ] Copy `CLAUDE.md` (standing conventions) into repo root
- [ ] Copy `BOILERPLATE.md` into repo root or `/docs`
- [ ] `composer run setup` = the complete, idempotent dev bootstrap
      (env-guarded against production; ends with `app:doctor`): install →
      hooks → .env/key-if-missing → sail services (postgres/redis/mailpit) →
      migrate --seed (seeder must be idempotent and seed the local
      dev@example.com developer) → npm install/build. README Path A becomes
      clone + setup + dev, where `composer run dev` brings up the Sail
      services (redis, mailpit) and runs the concurrent processes (Horizon is
      already the dev queue worker via Horizon's own `dev` integration — don't
      re-register it). Guard production two ways: a pre-install
      `getenv('APP_ENV')` shell check (exported env) plus an
      `app:ensure-development` artisan command after `.env` exists (boots the
      framework, so it also catches a `.env`-only production flag that
      getenv can't see)
- [ ] README "Production release" section: --no-dev optimized install,
      npm ci build, migrate --force, config/route/view/event cache,
      horizon:terminate; first-deploy AppSettingsSeeder +
      app:create-developer
- [ ] README with setup instructions for a new dev pulling this repo fresh —
      two explicit paths: hybrid (host PHP + `sail up -d redis mailpit` for
      services; the default) and full Sail (no host PHP; container hostnames
      in .env, everything via `./vendor/bin/sail`). State Docker Desktop as a
      prerequisite and end setup with `app:create-developer` + `app:doctor`
- [ ] Tag/release as `v1.0.0` of the boilerplate itself, so future
      boilerplate improvements are versioned and back-portable to projects
      that already forked it

---

## After this build

This file's job is done once Phase 9 is complete. `CLAUDE.md` takes over as
the file every future Claude Code session reads when working *inside* a
project generated from this boilerplate.
