# Laravel + Inertia SaaS Boilerplate

A reusable, opinionated starting point for a new Laravel SaaS app — optimized
for speed-to-first-user, with the architecture decided up front so it doesn't
drift from project to project.

> **Status:** planning docs. The boilerplate itself hasn't been scaffolded yet —
> this repo currently holds the architecture, the build plan, and the standing
> conventions. Run `BUILD_PLAN.md` to generate the actual starter.

## The three documents

Each file has a distinct role and lifecycle:

| File | Role | Lifecycle |
|---|---|---|
| **[BUILD_PLAN.md](BUILD_PLAN.md)** | The **generation** steps — phase-by-phase tasks to scaffold the repo from scratch. | Run once to build. Not shipped into generated projects. |
| **[BOILERPLATE.md](BOILERPLATE.md)** | The **architecture & why** — the reasoning behind every decision. | Copied into each generated project as the reference. |
| **[CLAUDE.md](CLAUDE.md)** | The **standing conventions** every Claude Code session (and human) follows when writing code inside a project. | Copied into each generated project's root; active on every session. |

In short: **BUILD_PLAN builds it, BOILERPLATE explains it, CLAUDE.md governs it afterward.**

## Stack

| Layer | Choice |
|---|---|
| Backend | Laravel (latest) |
| Authenticated UI | Inertia + Vue 3 + Pinia (TypeScript) |
| Public/marketing | Blade + Alpine.js |
| Admin | Filament (`/admin`, isolated) |
| Mobile/API | REST `/api/v1`, Sanctum token auth |
| Styling | Tailwind CSS |
| Auth | Fortify (password, 2FA), Socialite, `laravel/passkeys` |
| Validation / DTOs | Form Requests → `spatie/laravel-data` |
| Query params | `spatie/laravel-query-builder` |
| Feature flags | Laravel Pennant |
| Queues | Database driver (Redis/Horizon = upgrade path) |
| Payments | App-owned `PaymentGateway` interface (deferred) |

## Guiding principles

- **Shared service/action layer** — business logic lives in `app/Services` /
  `app/Actions`; web, API, and Filament controllers stay thin and call the
  same service, so rules can't drift between clients.
- **Strict typing, end to end** — `declare(strict_types=1)`, full type
  coverage, Larastan at `max`, typed DTOs over associative arrays, and Eloquent
  strict mode in dev. TypeScript on the frontend, with types generated from the
  same DTOs/Resources.
- **Convert at the boundary** — DTOs in, Resources out; UTC in the core, local
  at the edge; backend stays neutral and clients localize.
- **Idiomatic Laravel** — Eloquent scopes/relationships for query logic,
  Policies for authorization, events/observers for side effects; extend an
  existing pattern rather than introducing a new one.

## How to build from this plan

1. Read **BOILERPLATE.md** for the reasoning.
2. Work through **BUILD_PLAN.md** phase by phase — decide the starter-kit
   question in Phase 0 first, and run tests/lints between phases.
3. Copy **CLAUDE.md** and **BOILERPLATE.md** into the generated repo (Phase 9).

Deliberately deferred (documented, not built in v1): billing, error
monitoring, media/file uploads, real i18n infrastructure, and per-user
timezone/locale columns.
