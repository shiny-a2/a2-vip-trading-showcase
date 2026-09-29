# A2 — Trading-Signal Operations Platform (architecture showcase)

Latest maintenance work: [news resilience and membership boundary checks,
2026-09-29](docs/updates/2026-09-29-news-membership.md). The repair candidate is
tested offline; production deployment and incident verification are pending.

A production-grade platform for running a trading-signal operation end to end:
authoring and publishing signals, tracking each one to its outcome, memberships
and billing, performance reporting, referrals, support, and a signal-aware
assistant. This is an operations platform, not a single bot — several
long-running services share a common domain layer and speak to the same
database.

This repository is a **public-safe architecture showcase**. It documents the
design, the module boundaries, and the decisions behind the real system. It is
not the production source, and it contains no secrets, no host details, and no
provider or exchange names. See [What this repo is / isn't](#what-this-repo-is--isnt).

## Shape

The system is a modular monolith plus workers: a small set of supervised
processes that share well-bounded domain packages. Business logic lives in the
domain layer and in application services — never in message handlers or web
controllers. Dependencies point inward, toward the domain.

```
   Telegram Bot  ·  Mini App  ·  Admin Web  ·  HTTP API      (delivery)
                            |
                   Application Services                       (use cases, transactions)
                            |
                        Domain                                (entities, rules, invariants)
                            |
              Repositories & Adapters                         (Postgres, market data, messaging, LLM, chain)
```

## Services

| Component | Responsibility |
|-----------|----------------|
| API | HTTP backend for the Admin Web and Mini App; exposes application services |
| Telegram bot | Bot UX for signal entry, membership, and support (aiogram) |
| Worker | Market-data client, in-process price event bus, signal tracking, chart rendering, publishing pipeline, durable job consumer |
| Scheduler | Durable timed jobs: membership expiry, renewal reminders, reconciliation, reports, feed-health watchdog |
| Admin Web | Web admin panel (Next.js, work in progress) |
| Mini App | Messaging-platform mini application (Next.js, work in progress) |

All four backend services are native OS processes supervised by the operating
system's own task scheduler — boot-triggered, auto-restarting, no containers.
The rationale is recorded in the ADRs.

## Domain modules

The domain layer is split by bounded context. Each is a package under
`packages/domain`, exercised through application services:

- **identity** — users, admin accounts, authentication.
- **rbac** — granular, code-defined permissions and default role mappings, kept
  separate from membership tier. Sensitive operations are additionally gated to
  the owner at the service layer regardless of any granted permission.
- **billing** — plans, subscriptions, an entitlement engine derived from active
  subscriptions (never from a plan name), payment recording, and wallet-address
  configuration. Server-authoritative throughout.
- **signals** — the signal entity, its lifecycle state machine, R-multiple math,
  setup metrics, validation (level ordering, target arrays), symbols, and the
  tracking engine that turns price events into audited lifecycle events.
- **publishing** — the rules and records for pushing signals and their updates
  out to channels.
- **analytics** — track-record and R-report aggregation over closed signals.
- **audit** — an append-only ledger of every sensitive action (actor, action,
  before/after, reason, correlation id), not editable by ordinary admins.
- **referral**, **support**, **watchlist**, **content**, **ai** — the supporting
  contexts around the core signal and membership flows.
- **exchange** — the optional, execution-only integration layer (see below).

## Safety architecture

The design leads with a small number of hard guarantees, each enforced in code
rather than left to convention.

**The no-withdraw invariant.** The exchange integration is execution and
read-only. No adapter contains — and none may ever gain — a withdrawal or
transfer capability. The honest claim to a user is not "we checked your key" but
"our code physically cannot move your funds." This is enforced by an automated
invariant test that fails the build if any adapter grows a funds-movement method
or references a withdrawal path in its source. See
[ADR-0002](docs/adr/0002-no-withdraw-invariant.md).

**RBAC plus audit.** Authorization is a granular permission check separate from
membership tier. Every sensitive action writes an append-only audit record;
corrections are versioned with a reason and a second confirmation, not silent
edits.

**Server-authoritative billing.** The server never trusts the client for money.
Charges and entitlements are recomputed server-side. Payment activation is
idempotent by the provider's charge id, so a re-delivered event can never grant a
second subscription. On-chain USDT settlement is confirmed only against an
official indexer — never a user-submitted screenshot — with per-invoice unique
amounts so one transfer can never satisfy two invoices, and ambiguous transfers
routed to manual review rather than auto-credited.

**Migration discipline.** Schema changes go through reviewed, versioned
migrations; a production deploy requires green tests and a reviewed migration.

**Threat model.** An asset-centric, STRIDE-based threat model is maintained and
revisited as the system grows.

## Tech stack

- **Language / runtime:** Python 3.12, managed as a `uv` workspace (one
  virtualenv, path dependencies across packages).
- **Backend:** FastAPI (API), aiogram (bot), SQLAlchemy 2 async + PostgreSQL,
  Alembic migrations, Pydantic / pydantic-settings for config and schemas.
- **Async / infra:** httpx, websockets, APScheduler (in-process timer only),
  structlog for structured JSON logging.
- **Media:** matplotlib + Pillow for server-side chart rendering.
- **Security:** argon2-cffi (Argon2id password hashing), cryptography
  (encryption at rest), PyJWT.
- **Frontend:** Next.js for the Admin Web and Mini App (work in progress).
- **Quality gates:** ruff (lint + format), mypy in strict mode, pytest +
  pytest-asyncio, a secret scanner, and a migration check — all in CI.

No Redis in the first version: the durable job queue is a Postgres table consumed
with `SELECT … FOR UPDATE SKIP LOCKED` (at-least-once, idempotent handlers), the
scheduler is a polled Postgres table, and the price event bus is in-process
asyncio inside the single worker that owns the market-data socket. Each sits
behind a `Cache` / `Broker` / `EventBus` interface so a Redis backend can be
added later without touching call sites.

## What this repo is / isn't

**Is:** a sanitized description of the architecture — the service topology, the
domain boundaries, the safety invariants, and the recorded decisions. The code
sketches here are illustrative and written for this document.

**Isn't:** the production source. It carries no secrets, tokens, keys, wallet
addresses, or environment values; no host names, paths, or ports; and no
concrete provider or exchange names. Where the real system integrates a specific
vendor, this showcase describes the boundary and the interface, not the vendor.
