# ADR-0001: Modular monolith with supervised worker processes

- Status: Accepted

## Context

The platform spans a Python backend — an HTTP API, a messaging bot, a
market-data/publishing worker, and a scheduler — and a TypeScript frontend for
the admin panel and mini app. All of it depends on the same domain rules: what a
valid signal is, how a subscription grants entitlements, who may do what. If those
rules drift between a bot handler and a web controller, the system is wrong in a
way tests in any one service will not catch.

The deployment target has no container runtime and none is planned. Existing
services on the host already run as native processes supervised by the operating
system's task scheduler, which restarts them on boot and on crash. Introducing a
new always-on dependency (a separate broker, a cache server) means one more thing
that has to survive a reboot.

## Decision

- **A modular monolith plus workers.** One codebase, one domain layer, a small
  number of long-running processes that share it. Business logic lives in the
  domain and in application services; delivery code (bot handlers, HTTP routes,
  controllers) only authenticates, validates, and delegates. Dependencies point
  inward toward the domain.

- **A monorepo.** Python packages form a single `uv` workspace (one virtualenv,
  path dependencies); the JS apps form an npm workspace. Layout is `apps/*` for
  deployable units, `packages/*` for shared libraries, plus `infrastructure/*`,
  `docs/*`, `tests/*`, and `scripts/*`. Shared domain and contracts cannot drift
  between services because there is one copy.

- **Four backend processes.** An API, a bot, a worker, and a scheduler. The
  worker owns the market-data socket, the in-process price event bus, and the
  tracking engine together, because an in-process bus requires the consumer to
  live in the same process as the producer.

- **Postgres is the single source of truth; no second datastore in v1.** The
  durable job queue is a Postgres table consumed with `FOR UPDATE SKIP LOCKED`
  (at-least-once, idempotent handlers); the scheduler is a polled table; the price
  bus is in-process asyncio. Each sits behind a `Cache` / `Broker` / `EventBus`
  interface so a Redis backend can be introduced later without touching call
  sites.

- **Native-process supervision, no containers.** Each backend process is
  supervised by the host's task scheduler: start on boot, restart on failure. An
  idempotent install script is the source of truth for that wiring.

- **CI quality gates.** Lint, format check, strict type check, unit and
  integration tests, dependency audit, secret scan, and a migration check. A
  production deploy requires green tests and a reviewed migration.

## Consequences

- One domain layer keeps contracts consistent across every service and frontend;
  a rule is defined once.
- Fewer always-on dependencies to keep alive, secure, and monitor across a
  reboot — directly serving the reliability requirement.
- Cross-process pub/sub is not available in v1, so tracking is co-located with the
  market-data socket. This is acceptable at current scale and can be revisited by
  moving the bus behind its interface to a real broker.
- Isolation is achieved without containers: a dedicated database and role, a
  prefixed configuration namespace, a reserved port range, and a separate install
  root. If the platform later moves to a container host, the process boundaries
  are already drawn and supervision is swapped for restart policies.
