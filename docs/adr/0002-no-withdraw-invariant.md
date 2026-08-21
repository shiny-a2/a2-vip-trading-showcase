# ADR-0002: The no-withdraw invariant for exchange adapters

- Status: Accepted (execution stays behind a per-venue feature flag, off by
  default, enabled only after that venue's official API is integration-tested)

## Context

A user may connect their own exchange account so that signals can be executed on
their behalf, sized by their own capital and risk settings. To do that the
platform holds an API key for the user's account. That key is the most dangerous
asset in the system: if it could move money, a bug or a compromise could drain a
user's funds.

Two facts shape the decision. First, an exchange key can be issued without
withdrawal permission, and it should be — but not every venue exposes an endpoint
to verify a key's permissions after the fact, so "we checked your key has no
withdrawal rights" is not a guarantee we can always make truthfully. Second, the
integration only ever needs to read the account and place or cancel orders. It
never needs to withdraw or transfer anything.

The strongest honest guarantee is therefore not about the key's permissions. It
is about our code: our software physically cannot move a user's funds, because the
capability to do so does not exist anywhere in it.

## Decision

- **Adapters are execution- and read-only.** The exchange adapter interface
  exposes reads (price, balance, positions, order state) and execution (place,
  protect, cancel). It exposes no withdrawal, transfer, or internal-move method,
  and no adapter may add one.

- **The invariant is enforced by a test, not by convention.** An invariant test
  reflects over every adapter class and fails the build if any method name
  indicates a funds movement — `withdraw`, `transfer`, and similar — with a single
  permission-check method (`verify_no_withdraw`) explicitly allowed because it is
  a check, not a movement. A second test scans adapter source for withdrawal or
  transfer endpoint paths, as defense in depth against a movement hidden behind an
  innocuous method name. If anyone ever adds a funds-movement capability, the build
  goes red.

- **Keys are handled as secrets.** A connected key must carry no withdrawal
  permission by policy; it is encrypted at rest, never logged, and never exposed
  to the frontend or to the assistant. Where a venue supports it, an IP allowlist
  is recommended, and the key's permissions are verified before execution is
  allowed.

- **Execution is never fire-and-forget.** After every order the adapter confirms
  it registered, that the fill size is correct, that the stop is active, and that
  take-profit volumes do not exceed the position; on a partial fill, protective
  volumes are adjusted to the filled size. Every order carries a deterministic
  client order id, so a disconnect or timeout can never produce a duplicate;
  state is reconciled on reconnect before any retry.

- **Guards around execution.** Order preview before any order; global, per-user,
  and per-venue kill switches; a daily-loss guard; a max-order guard; a
  duplicate-order guard.

- **Off by default.** Live execution stays behind a per-venue feature flag,
  disabled until that venue's official, documented API is integration-tested and
  the owner turns it on. A read that cannot complete raises a transient error
  rather than returning an empty result, so a guard never reads "could not reach
  the venue" as "the account is flat."

## Consequences

- The custody claim to a user is simple and true regardless of what any given
  venue's API supports: the code cannot move their money.
- The guarantee cannot silently rot. A future contributor who adds a withdrawal
  method — for any reason — is stopped by a failing build, not by a code reviewer
  happening to notice.
- Execution is stateful by design: place, confirm, reconcile — not a single
  request. That is more work than fire-and-forget, and it is the correct amount of
  work for something that spends real money.
