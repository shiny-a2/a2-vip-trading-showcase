# Architecture

## 1. Goals and shape

A2 is a modular monolith plus workers: a handful of long-running processes that
share a set of well-bounded domain packages. Business logic lives in the domain
layer and in application services. Delivery code — bot handlers, web controllers,
HTTP routes — is thin: it authenticates, validates input, and calls a service.
Dependencies point inward toward the domain, and the domain depends on nothing
outside itself.

```
   Telegram Bot  ·  Mini App  ·  Admin Web  ·  HTTP API      (delivery layer)
                            |
                   Application Services                       (use cases, transactions)
                            |
                        Domain                                (entities, value objects, rules)
                            |
              Repositories & Adapters                         (Postgres, market data, messaging, LLM, chain)
```

## 2. Services (deployment units)

Four backend processes, each supervised by the operating system's task scheduler
so they start on boot and restart on crash. They share no code path with the
delivery-specific glue; each one is an entry point that wires the same
application services to a different transport.

| Process | Role |
|---------|------|
| API | REST backend for the Admin Web and Mini App; serves application services over HTTP |
| Telegram bot | aiogram bot; signal entry, membership, and support flows delegate to the same services |
| Worker | Market-data client, in-process price event bus, signal tracking engine, chart renderer, publishing pipeline, durable job consumer |
| Scheduler | Durable timed jobs: membership expiry and removal, renewal reminders, membership-drift reconciliation, reports, feed-health watchdog |

**Why the worker is one process.** The price event bus is in-process asyncio, so
the tracking engine has to live in the same process that holds the market-data
socket. Publishing and chart rendering are attached there to keep the whole
signal event path — tick in, lifecycle event out, caption updated — inside one
supervised unit. Heavier fan-out can be split behind the durable job queue later
without changing call sites.

The two web apps are static Next.js builds served behind a reverse proxy that
terminates TLS. They are work in progress.

## 3. Data flow — signal lifecycle

```
Admin authors a signal (Bot | Mini App | Admin Web)
  -> Application service validates: direction/level ordering, target array,
     entry-already-passed check
  -> Persist DRAFT/READY signal with its targets
  -> Chart render job -> image stored
  -> Publishing pipeline -> one message per channel, per publication rule

Market-data tick (WebSocket)
  -> Normalizer -> in-process price event bus
  -> Tracking engine detects entry / target / stop crossings per price-source policy
  -> Lifecycle event (idempotent, versioned, audited)
  -> Caption update + threaded reply on the published messages
  -> Analytics event
```

The signal state machine is explicit: transitions are enumerated and anything not
listed is illegal. Terminal states have no outgoing transitions.

```
DRAFT -> READY -> PUBLISHED -> WAITING_FOR_ENTRY -> ACTIVE -> TP_HIT* -> COMPLETED
                                                          \-> STOPPED -> COMPLETED
   (CANCELLED / INVALIDATED / EXPIRED / MANUALLY_CLOSED are terminal)
```

**Feed health.** If the price feed goes stale, tracking is marked degraded: no
automatic target or stop event is emitted, an admin alert fires, and REST
reconciliation with gap handling resumes tracking on recovery. When a gap makes
the order of a target versus a stop genuinely ambiguous, the event is recorded
for manual resolution rather than guessed. The system never invents a definitive
outcome from missing data.

**R-multiple math.** Outcomes are measured in units of initial risk,
`R = |entry − stop|`, computed with exact decimals. For a long, R grows as price
rises above entry; for a short, as price falls below entry; losses are negative
R. The same primitives feed the published caption and the analytics track record,
so the number a user sees and the number in the report are computed one way.

```python
from decimal import Decimal

def r_multiple(direction, entry: Decimal, stop: Decimal, price: Decimal) -> Decimal:
    """R achieved at `price`, in units of the initial risk |entry - stop|."""
    risk = abs(entry - stop)
    if risk == 0:
        raise ValueError("entry and stop must differ")
    return (price - entry) / risk if direction == "LONG" else (entry - price) / risk
```

## 4. Data flow — membership and billing

Billing is server-authoritative end to end. The client is never trusted for an
amount or an entitlement.

```
User buys a plan (in-app payment | on-chain USDT)
  -> In-app payment: authoritative only from a server-side payment event.
     Activation is idempotent by the provider's charge id, so a re-delivered
     event never grants a second subscription.
  -> USDT: an invoice is issued with a per-invoice unique amount. Settlement is
     confirmed only against an official chain indexer, bounded to transfers that
     arrived after the invoice was created. One transfer settles at most one
     invoice; ambiguous / under / overpaid transfers go to manual review, never
     auto-credit.
  -> grant_subscription: extend a same-tier active sub or open a new one, and
     write a permanent membership-ledger entry to the audit log
  -> Entitlements are recomputed from the user's ACTIVE subscriptions (unioned
     with the always-on free baseline) — derived from subscriptions, never from a
     plan name, and empty for a suspended user.
```

The separation matters: a plan is a sales artifact, an entitlement is a
capability. Access checks read the derived entitlement set, so changing what a
plan includes, or suspending a user, takes effect without touching call sites.

## 5. Module map

```
packages/
  config           typed settings from the environment
  observability    structured logging, metrics, correlation ids
  database         async SQLAlchemy engine/session, migration wiring
  domain/
    identity       users, admin accounts, authentication
    rbac           granular permissions, default role -> permission mapping
    billing        plans, subscriptions, entitlement engine, payments, wallets
    signals        entity, lifecycle state machine, R-math, metrics, validation,
                   symbols, tracking engine
    publishing     channel publication rules and records
    analytics      track record and R-report aggregation
    audit          append-only action ledger
    referral       referral graph and rewards
    support        ticketing
    watchlist      per-user watchlists
    content        managed bilingual content
    ai             signal-aware assistant service and prompts
    exchange       execution-only integration domain (adapter, execution,
                   verification, reconciliation, guards)
  security         password hashing, token issue/verify, field encryption
  i18n             translation keys; no user-facing string is hardcoded
  market-data      exchange market-data client and price normalization
  image-renderer   server-side chart rendering
  telegram         messaging transport helpers
  payments         chain-indexer client behind an interface
  exchanges        concrete, execution-only exchange adapters (wired in per flag)
apps/
  api  telegram-bot  worker  scheduler  admin-web  mini-app
```

## 6. Cross-cutting concerns

- **Config.** Typed settings come from the environment. Business and branding
  configuration lives in the database and is edited from the admin panel, not
  hardcoded.
- **i18n.** Every user-visible string is a translation key with values per
  locale; the platform is bilingual with one right-to-left and one left-to-right
  locale. No user-facing text is hardcoded.
- **Time.** UTC everywhere in storage and logic; local and Jalali conversion only
  at display.
- **Audit.** Every sensitive action writes an append-only record with actor,
  action, before/after, reason, and correlation id. Ordinary admins cannot edit
  it.
- **Observability.** Structured JSON logs that carry no secrets or clear PII,
  plus metrics and alerts.

## 7. The exchange-adapter boundary

The exchange integration is the sharpest boundary in the system, because it is
the only place that touches a user's funds at their own venue, and because it is
optional and off by default.

Design rules:

1. **Official APIs only.** Adapters speak an exchange's documented REST and
   WebSocket API. No reverse-engineered or private endpoints.
2. **One interface, many venues.** Every exchange implements the same `Protocol`,
   so the execution engine and its tests are venue-agnostic. A new venue is a new
   adapter, wired in only after its official API passes integration tests and the
   owner enables its feature flag. Until then the engine has no adapter for that
   venue and cannot place anything.
3. **Read and execute — never withdraw.** The interface exposes reads (price,
   balance, positions, order state) and execution (place, protect, cancel). It
   exposes no withdrawal or transfer. This is the no-withdraw invariant, and it
   is enforced by an automated test (ADR-0002).
4. **Never fire-and-forget.** Every order is confirmed after the fact — that it
   registered, that the fill size is right, that the stop is active, that
   take-profit volumes do not exceed the position. Every order carries a
   deterministic client order id so a disconnect or timeout never produces a
   duplicate; state is reconciled on reconnect before any retry.
5. **A read that fails is unknown, not empty.** A money-critical read that cannot
   complete raises a transient error rather than returning "nothing," so a guard
   never mistakes "could not read the account" for "the account is flat."

The abstract interface below is an illustrative sketch written for this document.
It shows the shape and the boundary; it is not the production code, and it names
no exchange.

```python
from __future__ import annotations

from dataclasses import dataclass
from decimal import Decimal
from typing import Any, Protocol, runtime_checkable


class TransientVenueError(Exception):
    """A read could not be completed (rate-limit / 5xx / non-200): the result is
    UNKNOWN, not empty. Money-critical readers raise this so a caller never treats
    'could not read' as 'confirmed flat'."""


@dataclass(frozen=True)
class PlacedOrder:
    exchange_order_id: str | None
    accepted: bool
    filled_qty: Decimal
    avg_price: Decimal | None


@dataclass(frozen=True)
class PositionInfo:
    symbol: str
    quantity: Decimal
    entry_price: Decimal | None
    stop_loss: Decimal | None = None
    take_profit: Decimal | None = None
    leverage: int | None = None


@runtime_checkable
class ExchangeAdapter(Protocol):
    """Execution- and read-only. There is deliberately NO withdraw/transfer method
    anywhere on this surface — the one thing an adapter is not allowed to do."""

    async def verify_no_withdraw(self, credential: object) -> bool:
        """Confirm the API key carries no withdrawal permission. Required before
        any execution is allowed. This is a permission CHECK, not a withdrawal."""
        ...

    # --- reads (some public, some credentialed; all side-effect free) ----------
    async def get_mark_price(self, symbol: str) -> Decimal | None: ...
    async def get_balance(self, credential: object) -> Decimal | None: ...
    async def get_position(self, credential: object, *, symbol: str) -> PositionInfo | None: ...
    async def list_positions(self, credential: object) -> list[PositionInfo]: ...

    # --- execution (place, protect, cancel) ------------------------------------
    async def place_order(
        self,
        credential: object,
        *,
        symbol: str,
        side: str,
        quantity: Decimal,
        client_order_id: str,   # deterministic -> a retry can never duplicate
        price: Decimal | None = None,
        reduce_only: bool = False,
    ) -> PlacedOrder: ...

    async def set_position_stop_loss(
        self, credential: object, *, symbol: str, price: Decimal
    ) -> bool: ...

    async def cancel_order(
        self, credential: object, *, symbol: str, exchange_order_id: str
    ) -> bool: ...

    # NOTE: no withdraw(), no transfer(), no move_funds(). The custody guarantee
    # is that this method does not — and can never — exist on an adapter.
```

The invariant is not merely documented; a test reflects over every adapter class
and fails the build if any method name suggests a funds movement (with the single
permission-check method explicitly allowed), and separately scans adapter source
for withdrawal or transfer endpoint paths. The full reasoning is in
[ADR-0002](docs/adr/0002-no-withdraw-invariant.md).

## 8. Isolation

The platform runs with its own dedicated database and least-privilege role, its
own prefixed configuration namespace, and a reserved set of network ports. It
shares no database, credential, session, or port with anything else on its host.
The runtime and isolation decisions are recorded in
[ADR-0001](docs/adr/0001-modular-monolith-services.md).
