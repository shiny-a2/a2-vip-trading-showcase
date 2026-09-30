# 0.1.1 — News resilience and membership notices

Status as of 2026-09-30: version 0.1.1 deployed; application service health verified.
Restored news delivery remains dependent on external API credit.

## What changed

- News ingestion isolates malformed responses, unavailable sources and timeouts.
  Healthy sources continue to supply articles when another source fails.
- Response validation tolerates malformed individual records and optional fields.
  Diagnostics omit credential-bearing URLs and raw provider exception text.
- Provider outages and credit restrictions pause rewriting with backoff without
  consuming each article's invalid-output retry budget.
- Expiry and renewal notices identify the affected plan in English or Persian,
  distinguish remaining valid subscriptions and independent operator permissions,
  and claim channel removal only when confirmed. Valid overlapping access remains
  effective.
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

A broken external source should not stop the whole news pipeline or exhaust
pending articles' retry budgets. Expiry of one plan does not necessarily end every
permission. Notices now describe that distinction, while delayed actions still
check current access and customers retain the ability to reduce existing exposure.

Live diagnostics identified an external API credit restriction blocking news
rewriting. They also confirmed that valid overlapping membership and independent
operator permissions can explain continued access after one plan expires. The
release preserves those valid permissions; it cannot restore external API credit.

## Verification boundary

The release passed 88 focused offline tests, including four Mini App scenarios.
Production-host Windows staging passed 87 tests; the JavaScript wrapper was skipped
there because its runtime was unavailable, and was verified locally. Database
integration tests were safely blocked by an unverified test schema. The checks use
mocked providers and adapters without placing trades or sending test messages.

The API, bot and scheduler were restarted and report the deployed release with
healthy database checks. The exchange worker continued running throughout the
update. A verified private backup and a tagged source release support rollback.
News recovery additionally requires restored external API credit and verified
fresh rewriting and publication.

Source code, customer records, operational access details and credentials remain
private.

## Staging update — 2026-09-29

An earlier candidate was uploaded to isolated Windows staging and passed all 53
focused offline tests there. The live service was unchanged by that upload. This
earlier validation preceded the final provider-backoff and notification changes;
it did not establish that the reported live incidents had been resolved.
