# Risk model

## Scoring

Score likelihood and impact 1–5 each. Risk = likelihood × impact, range 1–25.

**Likelihood**

| Score | Meaning |
|---|---|
| 5 | Almost certainly broken somewhere. Rewritten concurrent code, a migration with no dry run, an area that produced three defects this quarter. |
| 4 | Probably has a defect. Substantially changed logic with time/locale/money involved, or thin tests over a complex path. |
| 3 | Could plausibly break. Ordinary change to ordinary code with reasonable tests. |
| 2 | Unlikely. Small change, well-covered area, no shared state. |
| 1 | Very unlikely. Mechanical change — a rename, a comment, a version bump with no API change. |

**Impact**

| Score | Meaning |
|---|---|
| 5 | Irreversible or unbounded: data loss, money wrong, tenant isolation broken, reportable compliance breach. |
| 4 | Primary flow blocked for many users, or wrong output on a path users act on. |
| 3 | Secondary feature unusable, or a workaround exists but costs the user real effort. |
| 2 | Degraded experience, correct behaviour. |
| 1 | Cosmetic. |

**Bands**

| Risk | Depth | Rough share of the time box |
|---|---|---|
| 20–25 | Deep + failure injection + a rehearsed rollback | 40% |
| 15–19 | Deep | 30% |
| 8–14 | Standard | 20% |
| 4–7 | Shallow smoke | 10% |
| 1–3 | Out of scope, recorded | 0% |

The percentages are a starting allocation, not a rule. Their purpose is to make it
visible when a plan spends 70% of its time on risk-6 areas — which is what happens
by default, because low-risk areas are the easy, pleasant ones to test.

## Worked example

**Change:** a Spring Boot service adds idempotency keys to `POST /payments`.
Keys are stored in Redis with a 24-hour TTL; a duplicate key returns the original
response instead of charging again. Time box: three days before a release.

| # | Area | L | I | Risk | Depth | Level | Reasoning |
|---|---|---|---|---|---|---|---|
| 1 | Duplicate detection under concurrency | 5 | 5 | 25 | Deep | Integration | Two simultaneous requests with the same key is the exact case the feature exists for, and it is a check-then-act race. Money moves. |
| 2 | Redis unavailable / timeout | 4 | 5 | 20 | Deep | Integration + fault injection | New external dependency on the payment path. Fail-open double-charges; fail-closed blocks all payments. Both are S1, and the choice is a product decision that must be made explicitly. |
| 3 | TTL expiry boundary | 4 | 4 | 16 | Deep | Integration | Time-dependent logic. A retry at 24h ± seconds is exactly when a client's own retry policy fires. |
| 4 | Stored response replay correctness | 3 | 5 | 15 | Deep | Integration | Replaying a stale or wrong body returns a success for a payment that did not happen. Silent. |
| 5 | Key collision across tenants | 2 | 5 | 10 | Standard | Integration | Namespacing looks right in review, but the impact is cross-tenant data exposure, so verify rather than assume. |
| 6 | Existing payment flow without a key | 2 | 4 | 8 | Standard | Existing suite | Backward compatibility; old clients send no key. Covered by the current suite if it still passes. |
| 7 | Metrics and logging for the new path | 3 | 2 | 6 | Shallow | Manual | Observability matters, but a gap here is discovered and fixed in production cheaply. |
| 8 | Admin console payment list | 1 | 2 | 2 | None | — | Read-only, untouched by the change, no shared state. |

**Out of scope**

| Area | Why not | Accepted if wrong | Mitigation |
|---|---|---|---|
| Admin console (#8) | Not reachable from the change; read-only | A display bug for internal staff | None needed |
| Redis cluster failover | Infra-owned, no staging cluster available | Duplicate charges during a failover window | Feature flag; alert on `idempotency_lookup_error` rate; documented as a known gap in the release notes |
| Load beyond 200 rps | No load environment before the release date | Unknown latency added to the payment path | Canary at 5% traffic with p99 latency alert; rollback rehearsed |

**Highest risk:** #1, because it combines the highest likelihood signal
(check-then-act under concurrency) with the highest impact (money moves twice) and
the existing tests are single-threaded — so nothing currently in the suite could
catch it.

**Riskiest thing not tested:** Redis failover. The mitigation is a monitor and a
flag, not a test, and the team should agree to that in writing.

## Regression scope: what to re-run

"Run the full regression suite" is a way of avoiding the question. Scope it:

1. **Reachability** — what can the changed code actually reach? Follow the call graph
   out, and follow shared state (caches, static fields, DB tables, message topics)
   sideways. Shared state is the path people forget, and it is where the surprises are.
2. **Contract boundaries** — if a schema, payload, or file format changed, every
   consumer is in scope even if none of their code changed.
3. **Defect history** — areas that broke before break again. Weight them up
   independently of reachability.
4. **Configuration** — a change gated by a flag is two changes. Test the shipping
   combination, and the rollback combination.
5. **Everything else** — out of scope. Say so in the plan.

## Estimating without lying

When asked "how long will testing take", do not give a single number. Give three,
each attached to a stopping point:

- **Minimum** — the top two risk areas only. "We would know the payment path is safe;
  we would not know the admin views still render."
- **Planned** — everything down to risk 8. The recommended option.
- **Thorough** — down to risk 4, plus load and failover. Name what it costs in days.

This converts an argument about a number into a conversation about what the
business wants to know before it ships — which is the conversation worth having,
and the one that makes a QA engineer visible to the people who decide things.
