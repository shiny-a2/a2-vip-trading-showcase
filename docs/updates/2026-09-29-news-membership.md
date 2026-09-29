# News resilience and membership boundaries — 2026-09-29

Status: repair candidate prepared; production deployment and verification pending.

## What changed

- News ingestion isolates malformed responses, unavailable sources and timeouts.
  Healthy sources continue to supply articles when another source fails.
- Response validation tolerates malformed individual records and optional fields.
  Diagnostics omit credential-bearing URLs and raw provider exception text.
- A previously issued confirmation to increase position risk rechecks current
  execution permission. Expired membership or a trading halt preserves the existing
  protective stop instead of applying the requested widening.
- Cancellation of existing orders remains available through the intended safety
  path after membership expiry. The repair does not disable exchange credentials
  or remove protection from existing exposure.
- The Mini App refreshes membership when opening paid features. An expired account
  sees only the existing-position controls needed to close exposure.
- An anonymous read-only diagnostic helps distinguish news pipeline state, baseline
  plan access, overlapping memberships and operator exemptions.

## Why it matters

A broken external source should not stop the whole news pipeline. Membership
changes must be checked when an action is executed, including delayed confirmations,
while customers retain the ability to reduce existing exposure. The regression
checks use mocked providers and adapters without placing trades or sending messages.

## Verification boundary

The candidate passed 53 focused offline tests, including four Mini App scenarios,
plus syntax, lint and secret checks. Database integration and live checks remain pending.

The fixes address reproducible defects in the recovered deployment source. They
do not establish the cause of the reported live incidents. The candidate requires
comparison with the active checkout, runtime configuration review and production
health verification before it can be described as deployed or resolved. Source code,
customer records, operational access details and credentials remain private.
